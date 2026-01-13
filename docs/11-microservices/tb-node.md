# TB Node

## Overview

TB Node (ThingsBoard Node) is the core application server that serves as the central processing engine for the platform. It hosts the actor system, rule engine, REST API, and optionally transport protocols. TB Node can run as a monolith (all components together) or as a clustered microservice (multiple instances behind a load balancer).

## Architecture

```mermaid
graph TB
    subgraph "TB Node"
        subgraph "API Layer"
            REST[REST Controllers]
            WS[WebSocket Handlers]
        end

        subgraph "Processing Layer"
            AS[Actor System]
            RE[Rule Engine]
            DS[Device State Service]
        end

        subgraph "Service Layer"
            TS[Telemetry Service]
            ES[Entity Services]
            SS[Security Service]
        end

        subgraph "Queue Layer"
            QP[Queue Producers]
            QC[Queue Consumers]
        end
    end

    C[Clients] --> REST
    C --> WS
    REST --> AS
    WS --> AS
    AS --> RE
    AS --> DS
    RE --> TS
    RE --> ES
    TS --> QP
    QC --> AS
```

## Key Responsibilities

| Responsibility | Description |
|----------------|-------------|
| Actor System | Manages hierarchical actors for tenants, devices, rule chains |
| Rule Engine | Processes messages through configurable rule chains |
| REST API | Provides 50+ REST controllers for platform management |
| WebSocket | Handles real-time subscriptions and notifications |
| Queue Processing | Consumes and produces messages for distributed operation |
| Session Management | Tracks device sessions and connections |
| Security | Authenticates users and enforces permissions |

## Components

### Actor System

```mermaid
graph TB
    subgraph "Actor Dispatchers"
        AD[APP_DISPATCHER<br/>Pool: 1]
        TD[TENANT_DISPATCHER<br/>Pool: 2]
        DD[DEVICE_DISPATCHER<br/>Pool: 4]
        RD[RULE_DISPATCHER<br/>Pool: 8]
        CFM[CF_MANAGER_DISPATCHER<br/>Pool: 2]
        CFE[CF_ENTITY_DISPATCHER<br/>Pool: 8]
    end

    subgraph "Actor Hierarchy"
        AA[App Actor] --> TA[Tenant Actors]
        AA --> SA[Stats Actor]
        TA --> DA[Device Actors]
        TA --> RCA[Rule Chain Actors]
        TA --> CFA[Calculated Field Actors]
    end

    AD --> AA
    TD --> TA
    DD --> DA
    RD --> RCA
    CFM --> CFA
    CFE --> CFA
```

#### Dispatcher Configuration

| Dispatcher | Pool Size | Purpose |
|------------|-----------|---------|
| APP_DISPATCHER | 1 | Main application actor |
| TENANT_DISPATCHER | 2 | Tenant-level processing |
| DEVICE_DISPATCHER | 4 | Device session management |
| RULE_DISPATCHER | 8 | Rule engine execution |
| CF_MANAGER_DISPATCHER | 2 | Calculated field management |
| CF_ENTITY_DISPATCHER | 8 | Calculated field per-entity |

#### Actor Settings

| Setting | Default | Description |
|---------|---------|-------------|
| throughput | 5 | Messages per actor before switching |
| max_actor_init_attempts | 10 | Retries before disabling actor |
| scheduler_pool_size | 1 | Timer/scheduler thread pool |

### Rule Engine

```mermaid
flowchart TD
    subgraph "Rule Engine Processing"
        QI[Queue Input] --> RC[Rule Chain Actor]
        RC --> RN1[Rule Node 1]
        RN1 --> RN2[Rule Node 2]
        RN2 --> RN3[Rule Node 3]
        RN3 --> QO[Queue Output]
    end

    subgraph "Thread Pools"
        DBP[DB Callbacks: 50]
        MP[Mail: 40]
        SP[SMS: 50]
        EP[External: 50]
    end

    RN1 --> DBP
    RN2 --> MP
    RN2 --> SP
    RN3 --> EP
```

#### Rule Engine Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| db_callback_thread_pool_size | 50 | Database operation threads |
| mail_thread_pool_size | 40 | Email sending threads |
| sms_thread_pool_size | 50 | SMS sending threads |
| external_call_thread_pool_size | 50 | External HTTP call threads |
| transaction_queue_size | 15000 | Max queued transactions |
| transaction_duration | 60s | Transaction timeout |
| error_persist_frequency | 3s | Error logging interval |
| debug_rate_limit | 50000/hour | Debug events per tenant |

### REST API Layer

TB Node hosts 50+ REST controllers covering all platform operations:

```mermaid
graph LR
    subgraph "Controller Categories"
        EC[Entity Controllers<br/>Device, Asset, Dashboard...]
        DC[Data Controllers<br/>Telemetry, Attributes, Alarm]
        RC[RPC Controllers<br/>Device commands]
        AC[Admin Controllers<br/>Users, Tenants, Settings]
        SC[System Controllers<br/>Health, Info, Queues]
    end
```

#### Key API Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| rpc_min_timeout | 5s | Minimum RPC timeout |
| rpc_default_timeout | 10s | Default rule engine response timeout |
| max_payload_size | 16-52MB | Request body limit (varies by endpoint) |
| websocket_entity_limit | 10000 | Max entities per subscription |

### Queue Integration

```mermaid
sequenceDiagram
    participant TE as Transport/External
    participant Q as Message Queue
    participant TBN as TB Node
    participant DB as Database

    TE->>Q: Telemetry message
    Q->>TBN: Consume from tb_core
    TBN->>TBN: Process in rule engine
    TBN->>DB: Persist data
    TBN->>Q: Produce to tb_rule_engine
    Q->>TBN: Consume rule result
    TBN->>TE: Push notification
```

#### Queue Configuration

| Queue | Partitions | Poll Interval | Pack Timeout |
|-------|------------|---------------|--------------|
| Core (tb_core) | 10 | 25ms | 2s |
| Rule Engine | Varies | 25ms | 2s |
| Notifications | 1 | 25ms | 2s |
| Transport API | Varies | 25ms | 2s |

## Startup Sequence

```mermaid
sequenceDiagram
    participant M as Main
    participant SB as Spring Boot
    participant AS as Actor Service
    participant TS as Tenant Service
    participant PS as Partition Service

    M->>SB: Start application
    SB->>SB: Load configuration
    SB->>SB: Initialize beans

    Note over AS: @PostConstruct
    AS->>AS: Create actor system
    AS->>AS: Register dispatchers
    AS->>AS: Create AppActor
    AS->>AS: Create StatsActor

    Note over AS: @AfterStartUp
    AS->>AS: Send AppInitMsg
    AS->>TS: Load all tenants
    TS-->>AS: Tenant list
    AS->>AS: Initialize tenant actors

    PS->>PS: Resolve partitions
    PS->>PS: Broadcast startup
```

### Initialization Phases

| Phase | Actions |
|-------|---------|
| Configuration | Load thingsboard.yml, set up datasources |
| Bean Creation | Initialize Spring beans, auto-wiring |
| Actor Setup | Create actor system, dispatchers, root actors |
| Application Ready | Fire ApplicationReadyEvent, init tenant actors |
| Partition Discovery | Resolve queue partitions, broadcast to cluster |

## Clustering and Scaling

### Partition-Based Distribution

```mermaid
graph TB
    subgraph "Cluster"
        N1[TB Node 1<br/>Partitions 0-3]
        N2[TB Node 2<br/>Partitions 4-6]
        N3[TB Node 3<br/>Partitions 7-9]
    end

    subgraph "Shared Infrastructure"
        Q[Message Queue<br/>10 Partitions]
        DB[(Database)]
        ZK[Zookeeper<br/>Service Discovery]
    end

    N1 <--> Q
    N2 <--> Q
    N3 <--> Q
    N1 <--> DB
    N2 <--> DB
    N3 <--> DB
    N1 <--> ZK
    N2 <--> ZK
    N3 <--> ZK
```

### Cluster Communication

| Service | Method | Purpose |
|---------|--------|---------|
| pushMsgToCore | Queue | Core operations |
| pushMsgToRuleEngine | Queue | Rule processing |
| pushMsgToTransport | Queue | Transport notifications |
| pushMsgToEdge | Queue | Edge synchronization |
| broadcast | Queue | All-node announcements |

### Service Discovery

| Provider | Configuration |
|----------|---------------|
| Zookeeper | URL, connection timeout (3s), session timeout (3s) |
| None | Standalone mode (default) |

### Horizontal Scaling Strategy

```mermaid
flowchart TD
    LB[Load Balancer] --> N1[TB Node 1]
    LB --> N2[TB Node 2]
    LB --> N3[TB Node 3]

    subgraph "Shared State"
        Q[Kafka]
        DB[(PostgreSQL)]
        R[(Redis)]
    end

    N1 <--> Q
    N2 <--> Q
    N3 <--> Q
    N1 <--> DB
    N2 <--> DB
    N3 <--> DB
    N1 <--> R
    N2 <--> R
    N3 <--> R
```

## Configuration Reference

### Core Settings

| Property | Default | Description |
|----------|---------|-------------|
| service.type | monolith | monolith or tb-core |
| server.port | 8080 | HTTP port |
| server.ssl.enabled | false | Enable HTTPS |

### Actor System

| Property | Default | Description |
|----------|---------|-------------|
| actors.system.throughput | 5 | Messages per actor cycle |
| actors.system.scheduler_pool_size | 1 | Scheduler threads |
| actors.system.max_actor_init_attempts | 10 | Init retry limit |
| actors.tenant_dispatcher_pool_size | 2 | Tenant actor threads |
| actors.device_dispatcher_pool_size | 4 | Device actor threads |
| actors.rule_dispatcher_pool_size | 8 | Rule actor threads |

### Queue Settings

| Property | Default | Description |
|----------|---------|-------------|
| queue.type | in-memory | in-memory, kafka, aws-sqs, etc. |
| queue.core.partitions | 10 | Core queue partitions |
| queue.core.poll-interval | 25ms | Consumer poll interval |
| queue.core.pack-processing-timeout | 2s | Batch processing timeout |

### Transport (Monolith Mode)

| Property | Default | Description |
|----------|---------|-------------|
| transport.mqtt.enabled | true | Enable MQTT |
| transport.mqtt.bind_port | 1883 | MQTT port |
| transport.http.enabled | true | Enable HTTP |
| transport.coap.enabled | false | Enable CoAP |
| transport.sessions.inactivity_timeout | 600s | Session timeout |

## Deployment

### Docker Image

```mermaid
graph LR
    subgraph "tb-node Container"
        B[Base: Debian Slim]
        J[Java Runtime]
        T[ThingsBoard JAR]
        C[Configuration]
    end

    B --> J --> T --> C
```

### Directory Structure

| Path | Purpose |
|------|---------|
| /usr/share/thingsboard/bin/ | Application JAR |
| /usr/share/thingsboard/conf/ | Configuration files |
| /var/log/thingsboard/ | Log files |
| /config/ | External config mount point |

### Startup Modes

| Mode | Environment Variable | Action |
|------|----------------------|--------|
| Install | INSTALL_TB=true | Initialize database |
| Upgrade | UPGRADE_TB=true | Run migrations |
| Normal | (default) | Start application |

### Docker Compose Example

```yaml
tb-core1:
  image: thingsboard/tb-node:latest
  environment:
    - TB_SERVICE_ID=tb-core1
    - TB_SERVICE_TYPE=tb-core
    - TB_QUEUE_TYPE=kafka
    - TB_KAFKA_SERVERS=kafka:9092
    - ZOOKEEPER_ENABLED=true
    - ZOOKEEPER_URL=zookeeper:2181
  ports:
    - "8080:8080"
```

## Health and Monitoring

### Health Endpoints

| Endpoint | Purpose |
|----------|---------|
| /api/system/info | Build version, timestamp |
| /api/system/usage | Resource usage stats |
| /actuator/info | Spring Boot actuator info |

### Metrics and Statistics

| Metric Type | Configuration |
|-------------|---------------|
| Queue stats | Print interval: 60s |
| Actor stats | Persistence frequency: configurable |
| Rule engine stats | Persistence: 1 hour |
| Cluster stats | Disabled by default |

### Logging Configuration

| Component | Log File |
|-----------|----------|
| Application | /var/log/thingsboard/thingsboard.log |
| Rule Engine | Rule chain debug mode |
| Audit | Audit log service |

## Resource Requirements

### Memory Footprint

| Component | Typical Usage |
|-----------|---------------|
| Actor System | Varies by tenant count |
| Rule Engine | Transaction queue: 15000 msgs |
| Kafka Buffers | 32MB (configurable) |
| Session Cache | Per-device tracking |
| Partition Cache | 100,000 entries |

### Thread Pool Summary

| Pool | Threads | Purpose |
|------|---------|---------|
| App Dispatcher | 1 | App actor |
| Tenant Dispatcher | 2 | Tenant actors |
| Device Dispatcher | 4 | Device actors |
| Rule Dispatcher | 8 | Rule processing |
| DB Callbacks | 50 | Database operations |
| Mail Service | 40 | Email sending |
| SMS Service | 50 | SMS sending |
| External Calls | 50 | HTTP integrations |
| Transport API | 16+100 | Handler + callbacks |

### Scaling Recommendations

| Scale | Configuration |
|-------|---------------|
| Small (< 1000 devices) | Single node, in-memory queue |
| Medium (1000-10000 devices) | 2-3 nodes, Kafka queue |
| Large (> 10000 devices) | 3+ nodes, Kafka cluster, Redis cache |

## Common Patterns

### Monolith Deployment

```mermaid
graph TB
    subgraph "Single TB Node"
        REST[REST API]
        MQTT[MQTT Transport]
        HTTP[HTTP Transport]
        AS[Actor System]
        RE[Rule Engine]
        Q[In-Memory Queue]
    end

    D[Devices] --> MQTT
    D --> HTTP
    C[Clients] --> REST
```

### Microservices Deployment

```mermaid
graph TB
    subgraph "TB Core Cluster"
        N1[TB Node 1]
        N2[TB Node 2]
    end

    subgraph "Transport Services"
        M[MQTT Transport]
        H[HTTP Transport]
    end

    subgraph "Infrastructure"
        K[Kafka]
        P[(PostgreSQL)]
        R[(Redis)]
    end

    D[Devices] --> M
    D --> H
    M --> K
    H --> K
    K --> N1
    K --> N2
    N1 --> P
    N2 --> P
    N1 --> R
    N2 --> R
```

## Best Practices

### For Production Deployments

- Use Kafka for queue (not in-memory)
- Enable Redis for distributed caching
- Configure proper heap sizes based on load
- Set up health monitoring and alerts
- Use rolling deployments for zero downtime

### For Development

- Use monolith mode with in-memory queue
- Enable debug logging for rule engine
- Use smaller thread pool sizes
- Mount configuration externally for iteration

### For High Availability

- Deploy minimum 3 TB Node instances
- Use load balancer with health checks
- Configure partition distribution
- Enable Zookeeper for service discovery
- Monitor queue lag metrics

## See Also

- [Microservices Overview](./README.md) - Architecture overview
- [JS Executor](./js-executor.md) - JavaScript execution service
- [Transport Services](./transport-services.md) - Device transport services
- [Actor System Overview](../03-actor-system/README.md) - Actor details
- [Rule Engine Overview](../04-rule-engine/README.md) - Rule processing
