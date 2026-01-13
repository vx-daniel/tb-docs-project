# Message Queue Architecture

## Overview

The message queue provides asynchronous, decoupled communication between platform services. It enables horizontal scaling by distributing message processing across multiple service instances while maintaining message ordering through hash-based partitioning. The platform supports multiple queue providers (Kafka, In-Memory) and uses Protocol Buffers for efficient message serialization.

## Key Behaviors

1. **Service Decoupling**: Services communicate through message queues rather than direct calls, enabling independent scaling and fault isolation.

2. **Hash-Based Partitioning**: Messages are distributed to partitions using consistent hashing on entity IDs, ensuring all messages for an entity go to the same partition.

3. **Tenant Isolation**: Non-system tenants can have isolated queue topics, preventing noisy neighbor effects.

4. **Multiple Providers**: Supports Kafka for production deployments and in-memory queues for testing/single-node setups.

## Queue Factory Implementations

The platform uses factory patterns to support different queue providers:

| Factory | Implementation | Use Case |
|---------|----------------|----------|
| KafkaMonolithQueueFactory | Kafka-based queues | Production monolith |
| KafkaTbCoreQueueFactory | Kafka core queues | Microservices core |
| KafkaTbRuleEngineQueueFactory | Kafka rule engine | Microservices RE |
| KafkaTbTransportQueueFactory | Kafka transport | Microservices transport |
| KafkaEdqsQueueFactory | Kafka EDQS | Entity queries |
| InMemoryMonolithQueueFactory | In-memory queues | Development/testing |
| InMemoryTbTransportQueueFactory | In-memory transport | Single-node testing |

### Queue State Management

| Service | Description |
|---------|-------------|
| KafkaQueueStateService | Persisted queue offsets |
| DefaultQueueStateService | In-memory state tracking |
| QueueRoutingInfoService | Dynamic queue routing |

## Architecture Overview

```mermaid
graph TB
    subgraph "Producers"
        TRANS[Transport Services]
        CORE[Core Service]
        RE[Rule Engine]
    end

    subgraph "Message Queue Cluster"
        subgraph "Core Topics"
            TC[tb_core]
            TC_N[tb_core.notifications]
        end

        subgraph "Rule Engine Topics"
            TRE[tb_rule_engine]
            TRE_N[tb_rule_engine.notifications]
        end

        subgraph "Other Topics"
            TE[tb_edge]
            TT[tb_transport.notifications]
            TU[tb_usage_stats]
            TH[tb_housekeeper]
        end
    end

    subgraph "Consumers"
        CORE_C[Core Consumers]
        RE_C[Rule Engine Consumers]
        TRANS_C[Transport Consumers]
    end

    TRANS -->|telemetry, attributes| TC
    TRANS -->|rule processing| TRE
    CORE -->|notifications| TC_N
    RE -->|callbacks| TC

    TC --> CORE_C
    TRE --> RE_C
    TT --> TRANS_C
```

## Topic Structure

### Topic Naming Convention

```
[prefix.]<base_topic>[.isolated.<tenantId>][.<partition>]
```

| Component | Description | Example |
|-----------|-------------|---------|
| prefix | Optional environment prefix | `prod.` |
| base_topic | Core topic name | `tb_core` |
| isolated | Tenant isolation marker | `.isolated.` |
| tenantId | Tenant UUID for isolation | `a1b2c3d4-...` |
| partition | Partition number | `.0`, `.1` |

### Primary Topics

| Topic | Purpose | Default Partitions | Retention |
|-------|---------|-------------------|-----------|
| `tb_core` | Core platform messages | 10 | 7 days |
| `tb_core.notifications` | Direct core service notifications | 1 per service | 7 days |
| `tb_rule_engine` | Rule engine processing | Configurable | 7 days |
| `tb_rule_engine.notifications` | Direct rule engine notifications | 1 per service | 7 days |
| `tb_transport.notifications` | Transport service notifications | 1 per service | 7 days |
| `tb_transport.api.requests` | Transport API requests | 10 | 7 days |
| `tb_transport.api.responses` | Transport API responses | 10 | 7 days |
| `tb_edge` | Edge device synchronization | 10 | 7 days |
| `tb_edge.notifications` | Edge service notifications | 1 per service | 7 days |
| `tb_usage_stats` | Usage statistics collection | 1 | 7 days |
| `tb_ota_package` | OTA update distribution | 10 | 7 days |
| `tb_housekeeper` | Cleanup and maintenance tasks | 10 | 7 days |
| `tb_housekeeper.reprocessing` | Failed task reprocessing | 1 | 90 days |
| `tb_version_control` | Version control operations | 10 | 7 days |
| `tb_cf_event` | Calculated field events | Configurable | 7 days |
| `tb_cf_state` | Calculated field state | Configurable | 7 days |

### JavaScript Executor Topics

| Topic | Purpose | Default Partitions | Retention |
|-------|---------|-------------------|-----------|
| `js_eval.requests` | JavaScript evaluation requests | 30 | 1 day |
| `js_eval.responses.{serviceId}` | JavaScript evaluation responses | 1 per service | 1 day |

### EDQS Topics

| Topic | Purpose | Default Partitions | Retention |
|-------|---------|-------------------|-----------|
| `edqs.events` | Entity change events | 12 | 1 day |
| `edqs.state` | State snapshots | 12 | Infinite (compacted) |
| `edqs.requests` | Query requests | 12 | 3 minutes |
| `edqs.responses` | Query responses | 12 | 3 minutes |

### Task Queue Topics

| Topic | Purpose | Default Partitions | Retention |
|-------|---------|-------------------|-----------|
| `jobs.stats` | Task execution statistics | 1 | 7 days |
| `{job-type}` | Per-job-type task queues | 12 | 7 days |

### Notification Topics

Each service instance gets a dedicated notification topic for direct addressing:

```mermaid
graph LR
    subgraph "Core Service Instances"
        C1[core-1]
        C2[core-2]
        C3[core-3]
    end

    subgraph "Notification Topics"
        N1[tb_core.notifications.core-1]
        N2[tb_core.notifications.core-2]
        N3[tb_core.notifications.core-3]
    end

    C1 -.->|subscribes| N1
    C2 -.->|subscribes| N2
    C3 -.->|subscribes| N3

    OTHER[Other Service] -->|direct message| N2
```

## Message Format

### TbQueueMsg Interface

```
┌─────────────────────────────────────┐
│           TbQueueMsg                │
├─────────────────────────────────────┤
│ key: UUID                           │  → Partition routing key
│ headers: Map<String, byte[]>        │  → Metadata
│ data: byte[]                        │  → Protobuf payload
└─────────────────────────────────────┘
```

| Field | Type | Purpose |
|-------|------|---------|
| key | UUID | Used for partitioning and ordering |
| headers | Map<String, byte[]> | Metadata (producer ID, timestamps) |
| data | byte[] | Serialized Protocol Buffer message |

### Message Headers

| Header | Description |
|--------|-------------|
| producerId | Service instance that produced the message |
| ts | Timestamp when message was created |
| threadName | (Debug) Producer thread name |
| stackTrace | (Debug) Producer stack trace |

### Serialization

All message payloads use Protocol Buffers for efficient binary serialization:

```mermaid
graph LR
    OBJ[Java Object] -->|TbKafkaEncoder| PROTO[Protobuf]
    PROTO -->|serialize| BYTES[byte array]
    BYTES -->|TbKafkaDecoder| PROTO2[Protobuf]
    PROTO2 -->|deserialize| OBJ2[Java Object]
```

## Partitioning Strategy

### Hash-Based Partitioning

Messages are assigned to partitions using consistent hashing:

```
partition = abs(hash(entityId)) % partitionCount
```

```mermaid
graph TB
    MSG[Message with EntityId] --> HASH[Hash Function]
    HASH --> MOD[Modulo Partitions]
    MOD --> P0[Partition 0]
    MOD --> P1[Partition 1]
    MOD --> P2[Partition 2]
    MOD --> PN[Partition N]
```

### Supported Hash Functions

| Function | Description | Use Case |
|----------|-------------|----------|
| murmur3_128 | 128-bit Murmur3 hash (default) | Best distribution |
| murmur3_32 | 32-bit Murmur3 hash | Faster, less collision resistant |
| sha256 | Cryptographic SHA-256 | Security-sensitive contexts |

### Partition Configuration

| Queue | Config Property | Default |
|-------|-----------------|---------|
| Core | `queue.core.partitions` | 10 |
| Rule Engine | Per-queue configuration | Varies |
| Edge | `queue.edge.partitions` | 10 |
| Version Control | `queue.vc.partitions` | 10 |
| EDQS | `queue.edqs.partitions` | 12 |
| Tasks | `queue.tasks.partitions` | 12 |

### Partition Assignment

```mermaid
sequenceDiagram
    participant S1 as Service 1
    participant S2 as Service 2
    participant HPS as HashPartitionService
    participant Q as Queue

    Note over S1,Q: Service startup/cluster change

    S1->>HPS: Announce presence
    S2->>HPS: Announce presence
    HPS->>HPS: Calculate partition assignment

    HPS-->>S1: Assigned: [0, 1, 2, 3, 4]
    HPS-->>S2: Assigned: [5, 6, 7, 8, 9]

    S1->>Q: Subscribe to partitions 0-4
    S2->>Q: Subscribe to partitions 5-9
```

## Tenant Isolation

### Isolation Mechanism

Non-system tenants can have dedicated queue topics:

```mermaid
graph TB
    subgraph "System Tenant (Shared)"
        SYS[System Messages] --> SHARED[tb_rule_engine]
    end

    subgraph "Isolated Tenant A"
        TA[Tenant A Messages] --> ISO_A[tb_rule_engine.isolated.tenant-a-id]
    end

    subgraph "Isolated Tenant B"
        TB[Tenant B Messages] --> ISO_B[tb_rule_engine.isolated.tenant-b-id]
    end
```

### Topic Name Generation

| Tenant Type | Topic Pattern |
|-------------|---------------|
| System tenant | `tb_rule_engine.0` |
| Regular tenant | `tb_rule_engine.0` (shared) |
| Isolated tenant | `tb_rule_engine.isolated.<tenantId>.0` |

### TopicPartitionInfo

```
TopicPartitionInfo {
    topic: "tb_rule_engine"
    tenantId: <tenant-uuid>           // null for system
    partition: 5
    useInternalPartition: false
    myPartition: true                  // assigned to this service
    fullTopicName: "tb_rule_engine.isolated.<tenantId>.5"
}
```

## Service Types

### Service Type Enumeration

| Service Type | Label | Purpose |
|--------------|-------|---------|
| TB_CORE | TB Core | Core platform operations |
| TB_RULE_ENGINE | TB Rule Engine | Rule chain processing |
| TB_TRANSPORT | TB Transport | Device protocol handling |
| JS_EXECUTOR | JS Executor | JavaScript rule node execution |
| TB_VC_EXECUTOR | TB VC Executor | Version control operations |
| EDQS | TB Entity Data Query | Entity data queries |
| TASK_PROCESSOR | Task Processor | Background job processing |

### Service Communication Flow

```mermaid
graph TB
    subgraph "Transport Layer"
        MQTT[MQTT Transport]
        HTTP[HTTP Transport]
        COAP[CoAP Transport]
    end

    subgraph "Core Layer"
        CORE[TB Core]
    end

    subgraph "Processing Layer"
        RE[Rule Engine]
        JS[JS Executor]
    end

    MQTT -->|tb_core| CORE
    HTTP -->|tb_core| CORE
    COAP -->|tb_core| CORE

    CORE -->|tb_rule_engine| RE
    RE -->|js execution| JS
    JS -->|result| RE
    RE -->|callbacks| CORE

    CORE -->|tb_transport.notifications| MQTT
    CORE -->|tb_transport.notifications| HTTP
```

## Queue Providers

### Kafka Provider

Production-ready distributed message queue.

**Configuration:**
```yaml
queue:
  type: kafka
  kafka:
    bootstrap.servers: localhost:9092
    acks: all
    retries: 1
    compression.type: none
    batch.size: 16384
    linger.ms: 1
    max.request.size: 1048576
    replication_factor: 1
```

**Producer Settings:**

| Setting | Default | Description |
|---------|---------|-------------|
| acks | all | All replicas must acknowledge |
| retries | 1 | Retry count on failure |
| batch.size | 16384 | Batch size in bytes |
| linger.ms | 1 | Wait time for batching |
| compression.type | none | Message compression |
| buffer.memory | 33554432 | Producer buffer size |

**Consumer Settings:**

| Setting | Default | Description |
|---------|---------|-------------|
| auto.offset.reset | earliest | Start from earliest on new group |
| session.timeout.ms | 10000 | Session timeout |
| max.poll.records | 8192 | Max records per poll |
| max.poll.interval.ms | 300000 | Max poll interval |
| max.partition.fetch.bytes | 16777216 | Max fetch per partition |

### In-Memory Provider

For testing and single-node deployments.

**Configuration:**
```yaml
queue:
  type: in-memory
```

**Characteristics:**
- No persistence (data lost on restart)
- No network overhead
- Suitable for development/testing
- Single-node only

## Consumer Groups

### Group ID Format

```
[prefix]<serviceType><queueName>[-isolated-<tenantId>]-consumer[-<partitionId>]
```

**Examples:**
- `tb_rule_engine_main-consumer` - Shared main queue
- `tb_core_main-isolated-abc123-consumer-0` - Isolated tenant, partition 0

### Consumer Coordination

```mermaid
sequenceDiagram
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    participant C3 as Consumer 3
    participant Q as Queue (10 partitions)

    Note over C1,Q: Initial assignment
    C1->>Q: Join group
    Q-->>C1: Partitions [0,1,2,3]

    C2->>Q: Join group
    Note over Q: Rebalance
    Q-->>C1: Partitions [0,1,2]
    Q-->>C2: Partitions [3,4,5,6]

    C3->>Q: Join group
    Note over Q: Rebalance
    Q-->>C1: Partitions [0,1,2]
    Q-->>C2: Partitions [3,4,5]
    Q-->>C3: Partitions [6,7,8,9]

    Note over C2: Consumer 2 fails
    C2--xQ: Disconnect
    Note over Q: Rebalance
    Q-->>C1: Partitions [0,1,2,3,4]
    Q-->>C3: Partitions [5,6,7,8,9]
```

## Producer/Consumer Interfaces

### TbQueueProducer

```
interface TbQueueProducer<T extends TbQueueMsg> {
    getDefaultTopic(): String
    send(tpi: TopicPartitionInfo, msg: T, callback: TbQueueCallback): void
    stop(): void
}
```

### TbQueueConsumer

```
interface TbQueueConsumer<T extends TbQueueMsg> {
    getTopic(): String
    subscribe(): void
    subscribe(partitions: Set<TopicPartitionInfo>): void
    poll(durationInMillis: long): List<T>
    commit(): void
    unsubscribe(): void
    stop(): void
    isStopped(): boolean
    getPartitions(): Set<TopicPartitionInfo>
}
```

### Message Flow

```mermaid
sequenceDiagram
    participant P as Producer
    participant Q as Queue
    participant C as Consumer
    participant H as Handler

    P->>P: Create TbQueueMsg
    P->>Q: send(tpi, msg, callback)
    Q-->>P: callback.onSuccess()

    loop Polling
        C->>Q: poll(duration)
        Q-->>C: List<TbQueueMsg>
        C->>H: Process messages
        H-->>C: Done
        C->>Q: commit()
    end
```

## Queue Factory Pattern

### Factory Hierarchy

```mermaid
graph TB
    subgraph "Interfaces"
        TCF[TbCoreQueueFactory]
        TREF[TbRuleEngineQueueFactory]
        TVCF[TbVersionControlQueueFactory]
        TTF[TbTransportQueueFactory]
    end

    subgraph "Kafka Implementation"
        KMF[KafkaMonolithQueueFactory]
        KTCF[KafkaTbCoreQueueFactory]
        KTREF[KafkaTbRuleEngineQueueFactory]
        KTTF[KafkaTbTransportQueueFactory]
    end

    subgraph "In-Memory Implementation"
        IMMF[InMemoryMonolithQueueFactory]
        IMTTF[InMemoryTbTransportQueueFactory]
    end

    KMF -.->|implements| TCF
    KMF -.->|implements| TREF
    KMF -.->|implements| TVCF

    KTCF -.->|implements| TCF
    KTREF -.->|implements| TREF
    KTTF -.->|implements| TTF

    IMMF -.->|implements| TCF
    IMMF -.->|implements| TREF
    IMTTF -.->|implements| TTF
```

### Factory Selection

| Deployment | Queue Type | Factory |
|------------|------------|---------|
| Monolith | Kafka | KafkaMonolithQueueFactory |
| Monolith | In-Memory | InMemoryMonolithQueueFactory |
| Microservices (Core) | Kafka | KafkaTbCoreQueueFactory |
| Microservices (Rule Engine) | Kafka | KafkaTbRuleEngineQueueFactory |
| Microservices (Transport) | Kafka | KafkaTbTransportQueueFactory |

## Request-Response Pattern

### Synchronous Communication Over Async Queue

```mermaid
sequenceDiagram
    participant REQ as Requester
    participant RQ as Request Queue
    participant RSQ as Response Queue
    participant SVC as Service

    REQ->>RQ: Send request (with correlationId)
    REQ->>RSQ: Subscribe (filter by correlationId)

    RQ->>SVC: Deliver request
    SVC->>SVC: Process
    SVC->>RSQ: Send response (with correlationId)

    RSQ-->>REQ: Deliver response
    REQ->>REQ: Match by correlationId
```

### Request Template

```
DefaultTbQueueRequestTemplate {
    requestTimeout: Duration
    maxPendingRequests: int
    pendingRequests: Map<UUID, ResponseFuture>

    send(request): Future<Response>
    cleanup(): void  // Remove stale requests
}
```

## Cluster Events

### Partition Change Events

When cluster membership changes, services receive partition reassignment events:

```mermaid
graph TB
    CHANGE[Cluster Change] --> HPS[HashPartitionService]
    HPS --> CALC[Recalculate Partitions]
    CALC --> EVENT[PartitionChangeEvent]
    EVENT --> CONSUMERS[Update Consumer Subscriptions]
```

### Event Types

| Event | Trigger | Action |
|-------|---------|--------|
| ClusterTopologyChangeEvent | Service added/removed | Recalculate assignments |
| PartitionChangeEvent | Partition reassignment | Update subscriptions |
| ServiceListChangedEvent | Service list changed | Update routing tables |

## Performance Considerations

### Throughput Optimization

| Setting | Impact | Recommendation |
|---------|--------|----------------|
| batch.size | Higher = better throughput | Increase for high volume |
| linger.ms | Higher = better batching | Balance with latency |
| compression.type | Reduces network, increases CPU | Enable for high volume |
| max.poll.records | More records per poll | Tune based on processing time |

### Latency Optimization

| Setting | Impact | Recommendation |
|---------|--------|----------------|
| linger.ms | Lower = lower latency | Set to 0-1ms for real-time |
| acks | "1" = faster, less durable | Use "all" for safety |
| max.poll.records | Fewer = faster commits | Balance with throughput |

### Partition Count Guidelines

| Factor | Recommendation |
|--------|----------------|
| Consumer count | At least 1 partition per consumer |
| Throughput | More partitions = more parallelism |
| Ordering | Same entity always same partition |
| Overhead | Too many partitions = metadata overhead |

## Configuration Reference

### Core Queue Settings

```yaml
queue:
  type: kafka                    # kafka | in-memory
  prefix: ""                     # Optional topic prefix

  core:
    topic: tb_core
    partitions: 10
    ota.topic: tb_ota_package
    usage-stats-topic: tb_usage_stats
    housekeeper.topic: tb_housekeeper

  rule-engine:
    topic: tb_rule_engine

  edge:
    topic: tb_edge
    partitions: 10

  vc:
    topic: tb_version_control
    partitions: 10

  partitions:
    hash_function_name: murmur3_128
```

### Kafka-Specific Settings

```yaml
queue:
  kafka:
    bootstrap.servers: localhost:9092
    ssl.enabled: false
    replication_factor: 1

    # Producer settings
    acks: all
    retries: 1
    batch.size: 16384
    linger.ms: 1
    buffer.memory: 33554432

    # Consumer settings
    auto.offset.reset: earliest
    session.timeout.ms: 10000
    max.poll.records: 8192
    max.poll.interval.ms: 300000
```

## See Also

- [Kafka Configuration](./kafka-configuration.md) - Detailed Kafka settings
- [Processing Strategies](./processing-strategies.md) - Submit and retry strategies
- [Partitioning](./partitioning.md) - Partition strategies
- [System Overview](../01-architecture/system-overview.md) - Platform architecture
- [Actor System](../03-actor-system/README.md) - Message consumers
- [Transport Contract](../05-transport-layer/transport-contract.md) - Message producers
- [Rule Engine](../04-rule-engine/) - Rule processing consumers
