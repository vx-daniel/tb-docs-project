# TBMQ Architecture

## Overview

TBMQ is built on a horizontally scalable architecture using Java and proven open-source technologies. The broker is designed to be fault-tolerant with no single point of failure, where every node in a cluster is identical. This architecture enables millions of concurrent connections and millions of messages per second throughput.

## Core Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        MQTT[MQTT Clients]
        WS[WebSocket Clients]
    end

    subgraph "TBMQ Node"
        LISTENER[Listeners]
        AUTH[Authentication]
        ACL[Access Control]
        SESSION[Session Manager]
        SUB[Subscription Manager]
        MSG[Message Handler]
        PERSIST[Persistence Handler]
    end

    subgraph "Storage Layer"
        KAFKA[(Apache Kafka)]
        REDIS[(Redis)]
        PG[(PostgreSQL)]
    end

    MQTT --> LISTENER
    WS --> LISTENER
    LISTENER --> AUTH
    AUTH --> ACL
    ACL --> SESSION
    SESSION --> SUB
    SUB --> MSG
    MSG --> PERSIST
    PERSIST --> KAFKA
    PERSIST --> REDIS
    PERSIST --> PG
```

## Component Details

### Listeners

Listeners accept incoming MQTT connections over various transports:

| Listener | Port | Protocol | Security |
|----------|------|----------|----------|
| TCP | 1883 | MQTT | Plain |
| TLS | 8883 | MQTT | SSL/TLS |
| WebSocket | 8084 | MQTT over WS | Plain |
| WebSocket Secure | 8085 | MQTT over WSS | SSL/TLS |

### Authentication

Authentication verifies client identity before allowing connection:

```mermaid
graph LR
    CONNECT[CONNECT Packet] --> AUTH_TYPE{Auth Type}
    AUTH_TYPE -->|Basic| BASIC[Username/Password]
    AUTH_TYPE -->|X.509| CERT[Certificate Chain]
    AUTH_TYPE -->|JWT| TOKEN[Token Validation]
    AUTH_TYPE -->|SCRAM| SCRAM[Challenge Response]

    BASIC --> VERIFY[Verify Credentials]
    CERT --> VERIFY
    TOKEN --> VERIFY
    SCRAM --> VERIFY

    VERIFY -->|Success| CONNACK_OK[CONNACK Accept]
    VERIFY -->|Failure| CONNACK_FAIL[CONNACK Reject]
```

### Access Control (ACL)

ACL rules determine what topics clients can publish to or subscribe from:

| Rule Type | Scope | Example |
|-----------|-------|---------|
| Client ID | Per client | Allow clientA on topic/data |
| Username | Per user | Allow user1 on sensors/# |
| Certificate CN | Per certificate | Allow CN=device1 on devices/+ |

### Session Manager

Manages client sessions including persistence:

| Session Type | Behavior | Storage |
|--------------|----------|---------|
| Clean Session | Cleared on disconnect | Memory only |
| Persistent Session | Retained across disconnects | Redis |

### Subscription Manager

Handles topic subscriptions and message routing:

```mermaid
graph TB
    SUB[SUBSCRIBE] --> PARSE[Parse Topic Filter]
    PARSE --> VALIDATE[Validate ACL]
    VALIDATE --> STORE[Store Subscription]
    STORE --> WILDCARD{Wildcard?}
    WILDCARD -->|Yes| TRIE[Add to Topic Trie]
    WILDCARD -->|No| EXACT[Add to Exact Match]
    TRIE --> SUBACK[SUBACK]
    EXACT --> SUBACK
```

### Message Handler

Processes incoming PUBLISH messages and routes to subscribers:

```mermaid
sequenceDiagram
    participant PUB as Publisher
    participant BROKER as TBMQ
    participant KAFKA as Kafka
    participant SUB as Subscriber

    PUB->>BROKER: PUBLISH (QoS 1)
    BROKER->>BROKER: Find matching subscriptions
    BROKER->>KAFKA: Persist message
    BROKER->>PUB: PUBACK
    BROKER->>SUB: PUBLISH
    SUB->>BROKER: PUBACK
```

## Storage Architecture

### Apache Kafka

Kafka provides durable message storage and cluster coordination:

| Topic | Purpose | Retention |
|-------|---------|-----------|
| tbmq.msg.all | All published messages | Configurable |
| tbmq.msg.app.* | Application client messages | Configurable |
| tbmq.msg.retained | Retained messages | Infinite |
| tbmq.client.session | Session state | Compacted |
| tbmq.client.subscriptions | Subscription state | Compacted |

### Redis

Redis stores session data for device clients:

| Data Type | Key Pattern | Purpose |
|-----------|-------------|---------|
| Hash | session:{clientId} | Session metadata |
| List | msgs:{clientId} | Queued messages |
| Set | subs:{clientId} | Subscriptions |

### PostgreSQL

PostgreSQL stores broker configuration and client credentials:

| Table | Purpose |
|-------|---------|
| mqtt_client_credentials | Authentication credentials |
| application_shared_subscription | Shared subscription config |
| broker_settings | Broker configuration |

## Cluster Architecture

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[HAProxy/NLB]
    end

    subgraph "TBMQ Cluster"
        N1[Node 1]
        N2[Node 2]
        N3[Node 3]
    end

    subgraph "Kafka Cluster"
        K1[Broker 1]
        K2[Broker 2]
        K3[Broker 3]
    end

    subgraph "Redis Cluster"
        R1[Primary]
        R2[Replica 1]
        R3[Replica 2]
    end

    LB --> N1
    LB --> N2
    LB --> N3

    N1 <--> K1
    N2 <--> K2
    N3 <--> K3

    N1 --> R1
    N2 --> R1
    N3 --> R1
```

### Cluster Communication

Nodes communicate through Kafka for:
- Session state synchronization
- Subscription propagation
- Message routing between nodes

### Client Distribution

Clients are distributed across nodes via load balancer:

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Round Robin | Equal distribution | General purpose |
| Least Connections | Route to least loaded | Variable load |
| IP Hash | Same client same node | Session affinity |

## Message Flow

### Publish Flow (QoS 1)

```mermaid
sequenceDiagram
    participant C as Client
    participant N1 as Node 1
    participant K as Kafka
    participant N2 as Node 2
    participant S as Subscriber

    C->>N1: PUBLISH (QoS 1)
    N1->>K: Write to topic
    K-->>N1: Ack
    N1->>C: PUBACK
    K-->>N2: Consume message
    N2->>S: PUBLISH
    S->>N2: PUBACK
```

### Shared Subscription Flow

```mermaid
graph TB
    PUB[Publisher] --> BROKER[TBMQ]
    BROKER --> KAFKA[Kafka Topic]
    KAFKA --> CG[Consumer Group]
    CG --> S1[Subscriber 1]
    CG --> S2[Subscriber 2]
    CG --> S3[Subscriber 3]
```

## Scalability Design

### Horizontal Scaling

| Component | Scaling Method | Limit |
|-----------|----------------|-------|
| TBMQ Nodes | Add nodes behind LB | Unlimited |
| Kafka | Add brokers, partitions | Kafka limits |
| Redis | Cluster mode | Redis limits |
| PostgreSQL | Read replicas | Connection limits |

### Performance Tuning

| Parameter | Impact | Tuning |
|-----------|--------|--------|
| Kafka partitions | Parallelism | Match node count |
| Redis connections | Session throughput | Pool size |
| JVM heap | Memory for sessions | Based on connections |
| Netty threads | Connection handling | CPU cores * 2 |

## Fault Tolerance

### Node Failure

```mermaid
graph LR
    subgraph "Before Failure"
        LB1[LB] --> N1A[Node 1]
        LB1 --> N2A[Node 2]
        LB1 --> N3A[Node 3 - Failed]
    end

    subgraph "After Failover"
        LB2[LB] --> N1B[Node 1]
        LB2 --> N2B[Node 2]
        N3B[Node 3 - Restarting]
    end
```

- Load balancer detects failed node via health checks
- Clients reconnect to healthy nodes
- Session state restored from Redis/Kafka
- Queued messages redelivered

### Data Durability

| Data | Durability | Recovery |
|------|------------|----------|
| Messages | Kafka replication | Automatic |
| Sessions | Redis persistence | On reconnect |
| Subscriptions | Kafka compacted topic | Automatic |
| Credentials | PostgreSQL replication | Automatic |

## Security Architecture

### TLS Configuration

```mermaid
graph LR
    CLIENT[Client] -->|TLS Handshake| LB[Load Balancer]
    LB -->|TLS Passthrough| NODE[TBMQ Node]
    NODE -->|Verify| CERT[Certificate Store]
```

### Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant N as TBMQ
    participant DB as PostgreSQL

    C->>N: CONNECT (username, password)
    N->>DB: Query credentials
    DB-->>N: Credential record
    N->>N: Verify password hash
    N->>C: CONNACK (success/failure)
```

## Monitoring

### Metrics

| Category | Metrics |
|----------|---------|
| Connections | Active, total, rejected |
| Messages | Published, delivered, dropped |
| Subscriptions | Total, shared, by QoS |
| Kafka | Lag, throughput, errors |
| System | CPU, memory, network |

### Health Checks

| Endpoint | Purpose |
|----------|---------|
| /health | Overall broker health |
| /ready | Ready to accept connections |
| /metrics | Prometheus metrics |

## See Also

- [MQTT Features](./mqtt-features.md) - Protocol feature details
- [Message Queue](../08-message-queue/README.md) - Kafka architecture
- [Security](../09-security/README.md) - Security concepts
