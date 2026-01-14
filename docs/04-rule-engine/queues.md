# Rule Engine Queues

## Overview

ThingsBoard uses message queues to guarantee reliable message processing, handle traffic spikes, and maintain system stability under high load. The Rule Engine subscribes to queues on startup and polls for new messages. Queues are configured with submit strategies (how messages are dispatched) and processing strategies (how failures are handled).

## Key Behaviors

1. **Message Ordering**: Submit strategies control message ordering (sequential, burst, batch).

2. **Failure Handling**: Processing strategies define retry behavior on failures.

3. **Queue Isolation**: Tenants can have isolated queues for resource separation.

4. **Checkpointing**: Messages can be routed between queues using Checkpoint nodes.

5. **Default Queues**: Main, HighPriority, and SequentialByOriginator are pre-configured.

## Queue Architecture

```mermaid
graph TB
    subgraph "Message Sources"
        DEVICE[Device Messages]
        API[API Calls]
        INTERNAL[Internal Events]
    end

    subgraph "Queue Layer"
        MAIN[(Main Queue)]
        HP[(HighPriority Queue)]
        SEQ[(Sequential Queue)]
        CUSTOM[(Custom Queues)]
    end

    subgraph "Rule Engine"
        CONSUMER[Queue Consumer]
        PACK[Message Pack]
        CHAIN[Rule Chain]
    end

    DEVICE --> MAIN
    API --> MAIN
    INTERNAL --> MAIN

    MAIN --> CONSUMER
    HP --> CONSUMER
    SEQ --> CONSUMER
    CUSTOM --> CONSUMER

    CONSUMER --> PACK
    PACK --> CHAIN
```

## Queue Types

### Kafka (Production)

Distributed, durable message queue for production environments.

| Feature | Description |
|---------|-------------|
| Persistence | Messages stored on disk |
| Scalability | Horizontal scaling via partitions |
| Durability | Replication for fault tolerance |
| Throughput | High-volume message handling |

### In-Memory (Development)

Lightweight queue for development and small deployments.

| Feature | Description |
|---------|-------------|
| Speed | Fast in-memory operations |
| Simplicity | No external dependencies |
| Persistence | None - messages lost on restart |
| Use Case | Development, testing, demos |

## Queue Configuration

### UI Configuration

System administrators configure queues via **Settings → Queues**.

```mermaid
graph TB
    ADMIN[System Admin] --> SETTINGS[Settings]
    SETTINGS --> QUEUES[Queues Tab]
    QUEUES --> ADD[Add Queue]
    ADD --> CONFIG[Configure Settings]
    CONFIG --> SAVE[Save]
```

### Queue Settings

| Setting | Description |
|---------|-------------|
| Name | Queue identifier for logging and statistics |
| Submit strategy | How messages are submitted to rule engine |
| Processing strategy | How failures and timeouts are handled |
| Polling settings | Batch size and poll intervals |
| Custom properties | Provider-specific settings |

## Submit Strategies

Submit strategies control how messages from a queue pack are delivered to rule chains.

### Sequential by Originator

Messages processed one-by-one per entity. New message for Device A waits until previous Device A message is acknowledged.

```mermaid
sequenceDiagram
    participant Q as Queue
    participant RE as Rule Engine
    participant D as Device A

    Q->>RE: Message 1 (Device A)
    Note over RE: Processing...
    RE-->>Q: ACK
    Q->>RE: Message 2 (Device A)
    Note over RE: Processing...
    RE-->>Q: ACK
```

**Use Case:** When message order matters per device (e.g., state machines).

### Sequential by Tenant

Messages processed sequentially within each tenant.

```mermaid
sequenceDiagram
    participant Q as Queue
    participant RE as Rule Engine

    Q->>RE: Message 1 (Tenant A)
    Note over RE: Processing...
    RE-->>Q: ACK
    Q->>RE: Message 2 (Tenant A)
```

**Use Case:** Tenant-level consistency requirements.

### Sequential (Global)

All messages processed one at a time globally.

**Use Case:** Strict ordering, but very slow throughput.

### Burst

All messages submitted immediately in arrival order.

```mermaid
sequenceDiagram
    participant Q as Queue
    participant RE as Rule Engine

    Q->>RE: Message 1, 2, 3, 4, 5 (parallel)
    Note over RE: Processing in parallel
```

**Use Case:** Maximum throughput, no ordering requirements.

### Batch

Messages grouped into batches by size. New batch waits for previous batch acknowledgment.

```mermaid
sequenceDiagram
    participant Q as Queue
    participant RE as Rule Engine

    Q->>RE: Batch 1 (messages 1-100)
    Note over RE: Processing batch...
    RE-->>Q: ACK Batch 1
    Q->>RE: Batch 2 (messages 101-200)
```

**Use Case:** Balanced throughput with controlled parallelism.

### Strategy Comparison

| Strategy | Ordering | Throughput | Use Case |
|----------|----------|------------|----------|
| Sequential by Originator | Per entity | Medium | Device state tracking |
| Sequential by Tenant | Per tenant | Medium | Tenant consistency |
| Sequential | Global | Low | Strict ordering |
| Burst | None | High | High throughput |
| Batch | Per batch | Medium-High | Balanced processing |

## Processing Strategies

Processing strategies control how failed or timed-out messages are handled.

### Retry Failed and Timeout

Retry all failed and timed-out messages from the processing pack.

```mermaid
graph TB
    PROCESS[Process Pack] --> CHECK{All Success?}
    CHECK -->|Yes| ACK[Acknowledge]
    CHECK -->|No| RETRY{Retry Allowed?}
    RETRY -->|Yes| WAIT[Wait Period]
    WAIT --> RESUBMIT[Resubmit Failed + Timed Out]
    RESUBMIT --> PROCESS
    RETRY -->|No| ACK
```

**Use Case:** Critical data requiring reliable delivery.

### Skip All Failures

Ignore failures, acknowledge messages anyway.

```mermaid
graph TB
    PROCESS[Process Pack] --> CHECK{Any Failures?}
    CHECK -->|Yes| LOG[Log Warning]
    CHECK -->|No| ACK[Acknowledge]
    LOG --> ACK
```

**Use Case:** Non-critical data, backward compatibility.

### Skip All Failures and Timeouts

Ignore both failures and timeouts, cancel timed-out messages.

**Use Case:** Best-effort processing, development environments.

### Retry All

Retry entire pack if any message fails.

```mermaid
graph TB
    PROCESS[Process 100 Messages] --> CHECK{Any Failure?}
    CHECK -->|1 Failed| CANCEL[Cancel All]
    CANCEL --> RETRY[Retry All 100]
    RETRY --> PROCESS
    CHECK -->|All Success| ACK[Acknowledge]
```

**Use Case:** Atomic processing requirements.

### Retry Failed

Retry only failed messages, not timed-out ones.

**Use Case:** Separate failure and timeout handling.

### Retry Timeout

Retry only timed-out messages, not failed ones.

**Use Case:** Network-sensitive workloads.

### Retry Configuration

| Parameter | Description |
|-----------|-------------|
| Number of retries | Maximum retry attempts (0 = unlimited) |
| Failure percentage | Skip retry if failures < X% of messages |
| Retry within | Initial wait time before retry (seconds) |
| Additional retry within | Wait time for subsequent retries |

### Strategy Comparison

| Strategy | On Failure | On Timeout | Use Case |
|----------|------------|------------|----------|
| Retry Failed and Timeout | Retry | Retry | Critical data |
| Skip All Failures | Skip | Process | Non-critical |
| Skip All Failures and Timeouts | Skip | Cancel | Best effort |
| Retry All | Retry all | Retry all | Atomic batches |
| Retry Failed | Retry | Skip | Failure focus |
| Retry Timeout | Skip | Retry | Network issues |

## Polling Settings

### Batch Processing

| Setting | Description |
|---------|-------------|
| Poll interval | Milliseconds between polls when no messages |
| Partitions | Number of parallel partitions |

### Immediate Processing

| Setting | Description |
|---------|-------------|
| Consumer per partition | Separate consumer per partition |
| Processing within | Timeout for pack processing (ms) |

## Default Queues

### Main Queue

| Setting | Value |
|---------|-------|
| Submit strategy | Burst |
| Processing strategy | Skip All Failures |
| Purpose | General message processing |

### HighPriority Queue

| Setting | Value |
|---------|-------|
| Submit strategy | Burst |
| Processing strategy | Retry Failed and Timeout |
| Purpose | Critical alerts, alarms |

### SequentialByOriginator Queue

| Setting | Value |
|---------|-------|
| Submit strategy | Sequential by Originator |
| Processing strategy | Retry Failed and Timeout |
| Purpose | Ordered device messages |

## Checkpoint Node

Use the **Checkpoint** node to route messages to different queues.

```mermaid
graph LR
    INPUT[Telemetry] --> FILTER{Critical?}
    FILTER -->|Yes| CHECKPOINT[Checkpoint: HighPriority]
    FILTER -->|No| SAVE[Save Telemetry]
    CHECKPOINT --> ACK[Original Queue ACK]
```

### Configuration

| Setting | Description |
|---------|-------------|
| Queue name | Target queue for message |

### Behavior

1. Message submitted to target queue
2. Original message acknowledged
3. Processing continues in target queue

## Isolated Tenant Queues

### Enable Isolation

Configure in tenant profile:

```mermaid
graph TB
    PROFILE[Tenant Profile] --> ENABLE[Enable Isolated Queues]
    ENABLE --> CONFIG[Configure Queues]
    CONFIG --> APPLY[Apply to Tenant]
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Resource isolation | Dedicated queue resources |
| Performance | No interference from other tenants |
| Customization | Custom queue settings per tenant |
| Security | Queue-level separation |

### Configuration

Per-tenant queues are configured in the tenant profile with the same settings as common queues.

## Custom Properties

Provider-specific queue properties:

**Kafka:**
```
retention.ms:604800000;retention.bytes:1048576000
```

**AWS SQS:**
```
MaximumMessageSize:262144;MessageRetentionPeriod:604800
```

**Note:** Properties apply only on queue creation.

## Monitoring

### Rule Engine Statistics Dashboard

Monitor queue processing via the built-in dashboard:

| Metric | Description |
|--------|-------------|
| Messages processed | Total messages per queue |
| Processing time | Average processing duration |
| Failures | Failed message count |
| Retries | Retry attempts |
| Queue depth | Pending messages |

## Best Practices

### Queue Selection

| Scenario | Recommended Queue |
|----------|-------------------|
| General telemetry | Main |
| Alarm notifications | HighPriority |
| State tracking | SequentialByOriginator |
| Critical operations | Custom with retry |

### Performance Tuning

| Practice | Benefit |
|----------|---------|
| Use Burst for high throughput | Maximum parallelism |
| Set appropriate partitions | Horizontal scaling |
| Configure retry limits | Prevent infinite loops |
| Monitor queue depth | Detect bottlenecks |

### Reliability

| Practice | Benefit |
|----------|---------|
| Use retry strategies for critical data | Guaranteed delivery |
| Set failure percentage threshold | Prevent cascade failures |
| Configure reasonable timeouts | Handle slow operations |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Messages not processed | Queue not started | Check queue configuration |
| High latency | Too few partitions | Increase partitions |
| Message loss | Skip strategy | Switch to retry strategy |
| Infinite retries | No failure threshold | Set failure percentage |
| Memory issues | Large queue depth | Scale consumers |

### Debug Steps

1. Check queue status in admin UI
2. Review Rule Engine statistics dashboard
3. Examine server logs for queue errors
4. Verify tenant queue assignment
5. Test with debug mode enabled

## Common Pitfalls

### Submit Strategy Selection

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Using BURST for ordered data | Messages processed out of sequence, state machine errors | Use SEQUENTIAL_BY_ORIGINATOR for state-dependent processing |
| SEQUENTIAL for high volume | Low throughput, bottleneck | Use BATCH or BURST with idempotent processing |
| Wrong batch size | Memory pressure or high latency | Start with 100-500 messages per batch; tune based on load |
| SEQUENTIAL_BY_TENANT for multi-tenant | One tenant blocks others | Use SEQUENTIAL_BY_ORIGINATOR or BURST for tenant isolation |

### Processing Strategy Configuration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| RETRY_ALL on external failures | Infinite retry loops on persistent errors | Set `failurePercentage` threshold (e.g., 10%) to stop retries |
| No retry limit | Queue backup on persistent failure | Set reasonable retry count (3-5 attempts typical) |
| SKIP_ALL_FAILURES | Data loss without notification | Use RETRY_FAILED_AND_TIMEOUT with alerting on repeated failures |
| Timeout too short | False failures on legitimate slow operations | Set timeout ≥ 2x expected duration; monitor 99th percentile latency |
| Timeout too long | Blocking slow operations | Set reasonable upper bound; use separate queue for slow operations |

### Queue Assignment and Configuration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| All chains in Main queue | Critical messages blocked by non-critical | Use HighPriority queue for critical chains |
| No dedicated queue for slow operations | Main queue blocked | Create separate queue for external API calls, heavy processing |
| Wrong queue for message type | Ordering issues or performance problems | Align queue strategy with message requirements |
| Queue depth not monitored | Queue overflow unnoticed | Monitor queue depth; alert on thresholds |
| No consumer scaling | Queue backup under load | Scale consumers horizontally; monitor processing rate |

### Partition Strategy

| Pitfall | Impact | Solution |
|---------|--------|----------|
| No partition key | Random distribution, no ordering | Set partition key (originator ID, tenant ID) for ordered processing |
| High-cardinality partition key | Partition imbalance | Use keys with reasonable cardinality (device/tenant, not message ID) |
| Partition key collision | Hot partitions | Choose keys with even distribution |
| Changing partition strategy | Message reordering during transition | Plan migration carefully; drain old configuration first |

### Retry and Error Handling

| Pitfall | Impact | Solution |
|---------|--------|----------|
| No failure path in rule chain | Messages retried indefinitely | Add Failure relation handlers; use ACK node to stop retries |
| Retrying non-idempotent operations | Duplicate actions (alarms, notifications) | Use idempotent operations or deduplication |
| No exponential backoff | Overwhelming failing external service | Implement backoff: delay × retryAttempt |
| Ignoring failure percentage | One bad message blocks queue | Set failure threshold; move to DLQ after threshold |

### Performance and Scaling

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Single consumer per queue | Underutilized resources | Scale consumers based on queue depth and processing rate |
| Too many partitions | Overhead, complexity | Start with 2-4 partitions per consumer; scale as needed |
| Batch size too small | High overhead | Increase batch size for throughput; decrease for latency |
| No backpressure handling | Memory exhaustion | Monitor queue depth; implement rate limiting if needed |
| Mixed priorities in one queue | Critical messages delayed | Use separate queues for different priorities |

### Configuration Mistakes

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Queue name typo | Messages go to wrong queue | Validate queue names; use constants |
| Queue not started | Messages not processed | Verify queue status in admin UI |
| Incorrect tenant assignment | Security boundary violation | Validate tenant-queue mapping |
| Partition count mismatch | Uneven distribution | Ensure partition count consistent across cluster |

## See Also

- [Rule Engine Overview](./README.md) - Rule engine architecture
- [Message Flow](./message-flow.md) - Message routing
- [Flow Nodes](./nodes/flow-nodes.md) - Checkpoint node
- [Message Queue](../08-message-queue/README.md) - Queue infrastructure
- [Rate Limiting](../09-security/rate-limiting.md) - Tenant rate limits
