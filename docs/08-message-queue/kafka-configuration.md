# Kafka Configuration

## Overview

Apache Kafka is the recommended message queue for production ThingsBoard deployments. This document details all Kafka-specific configuration options, including producer and consumer settings, SSL/TLS encryption, Confluent Cloud integration, topic configuration, and consumer statistics monitoring.

## Connection Settings

### Bootstrap Servers

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.bootstrap.servers` | Required | Comma-separated list of Kafka broker addresses |

```yaml
queue:
  kafka:
    bootstrap.servers: "kafka1:9092,kafka2:9092,kafka3:9092"
```

### Request Timeout

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.request.timeout.ms` | 30000 | Request timeout in milliseconds |
| `queue.kafka.session.timeout.ms` | 10000 | Consumer session timeout |

## SSL/TLS Configuration

### Enable SSL

```yaml
queue:
  kafka:
    ssl:
      enabled: true
      truststore:
        location: "/path/to/truststore.jks"
        password: "truststore-password"
      keystore:
        location: "/path/to/keystore.jks"
        password: "keystore-password"
      key:
        password: "key-password"
```

### SSL Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.ssl.enabled` | false | Enable SSL/TLS encryption |
| `queue.kafka.ssl.truststore.location` | (empty) | Path to truststore file |
| `queue.kafka.ssl.truststore.password` | (empty) | Truststore password |
| `queue.kafka.ssl.keystore.location` | (empty) | Path to keystore file |
| `queue.kafka.ssl.keystore.password` | (empty) | Keystore password |
| `queue.kafka.ssl.key.password` | (empty) | Private key password |

## Confluent Cloud Integration

For managed Kafka services like Confluent Cloud:

```yaml
queue:
  kafka:
    use_confluent_cloud: true
    confluent:
      ssl.algorithm: "https"
      sasl.mechanism: "PLAIN"
      sasl.config: "org.apache.kafka.common.security.plain.PlainLoginModule required username='API_KEY' password='API_SECRET';"
      security.protocol: "SASL_SSL"
```

### Confluent Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.use_confluent_cloud` | false | Enable Confluent Cloud support |
| `queue.kafka.confluent.ssl.algorithm` | (empty) | SSL algorithm |
| `queue.kafka.confluent.sasl.mechanism` | (empty) | SASL mechanism (PLAIN, SCRAM-SHA-256) |
| `queue.kafka.confluent.sasl.config` | (empty) | JAAS configuration string |
| `queue.kafka.confluent.security.protocol` | (empty) | Security protocol |

## Producer Configuration

### Core Producer Settings

```yaml
queue:
  kafka:
    acks: all
    retries: 1
    compression.type: none
    batch.size: 16384
    linger.ms: 1
    max.request.size: 1048576
    max.in.flight.requests.per.connection: 5
    buffer.memory: 33554432
```

### Producer Settings Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.acks` | all | Acknowledgment level (0, 1, all) |
| `queue.kafka.retries` | 1 | Number of retries on failure |
| `queue.kafka.compression.type` | none | Compression codec (none, gzip, snappy, lz4, zstd) |
| `queue.kafka.batch.size` | 16384 | Batch size in bytes |
| `queue.kafka.linger.ms` | 1 | Wait time for batching (ms) |
| `queue.kafka.max.request.size` | 1048576 | Maximum request size (1 MB) |
| `queue.kafka.max.in.flight.requests.per.connection` | 5 | Unacknowledged requests per connection |
| `queue.kafka.buffer.memory` | 33554432 | Total producer buffer memory (32 MB) |

### Acknowledgment Levels

```mermaid
graph LR
    subgraph "acks=0"
        P0[Producer] -->|Fire & forget| B0[Broker]
    end

    subgraph "acks=1"
        P1[Producer] -->|Wait for leader| B1[Leader]
        B1 -->|ACK| P1
    end

    subgraph "acks=all"
        P2[Producer] -->|Wait for ISR| B2[Leader]
        B2 -->|Replicate| R[Replicas]
        R -->|ACK| B2
        B2 -->|ACK| P2
    end
```

| Value | Durability | Latency | Description |
|-------|------------|---------|-------------|
| 0 | None | Lowest | No acknowledgment |
| 1 | Leader only | Medium | Leader acknowledgment |
| all | Full ISR | Highest | All in-sync replicas |

### Compression Options

| Type | CPU | Compression Ratio | Use Case |
|------|-----|-------------------|----------|
| none | None | 1:1 | Low latency |
| gzip | High | Best | Bandwidth constrained |
| snappy | Low | Good | Balanced |
| lz4 | Very Low | Good | High throughput |
| zstd | Medium | Excellent | Best compression |

## Consumer Configuration

### Core Consumer Settings

```yaml
queue:
  kafka:
    auto_offset_reset: earliest
    max_poll_records: 8192
    max_poll_interval_ms: 300000
    max_partition_fetch_bytes: 16777216
    fetch_max_bytes: 134217728
```

### Consumer Settings Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.auto_offset_reset` | earliest | Offset reset behavior |
| `queue.kafka.max_poll_records` | 8192 | Maximum records per poll |
| `queue.kafka.max_poll_interval_ms` | 300000 | Maximum poll interval (5 min) |
| `queue.kafka.max_partition_fetch_bytes` | 16777216 | Max bytes per partition (16 MB) |
| `queue.kafka.fetch_max_bytes` | 134217728 | Max fetch bytes (128 MB) |
| `queue.kafka.session.timeout.ms` | 10000 | Session timeout (10 sec) |

### Auto Offset Reset Options

| Value | Behavior |
|-------|----------|
| earliest | Start from beginning of topic |
| latest | Start from end of topic |
| none | Throw error if no offset |

### Per-Topic Consumer Properties

Override consumer settings for specific topics:

```yaml
queue:
  kafka:
    consumer-properties-per-topic:
      tb_ota_package:
        max.poll.records: 10
      tb_version_control:
        max.poll.records: 100
      tb_edge:
        max.poll.records: 100
      tb_housekeeper:
        max.poll.records: 100
```

## Topic Configuration

### Replication Factor

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.replication_factor` | 1 | Topic replication factor |

For production, set to at least 3:

```yaml
queue:
  kafka:
    replication_factor: 3
```

### Topic Properties

Configure topic-specific settings inline:

```yaml
queue:
  kafka:
    topic-properties:
      core: "retention.ms:604800000;segment.bytes:52428800"
      rule-engine: "retention.ms:604800000;segment.bytes:52428800"
      transport-api: "retention.ms:604800000;segment.bytes:52428800"
      notifications: "retention.ms:604800000;segment.bytes:52428800"
      js-executor: "retention.ms:86400000;segment.bytes:52428800"
      ota-updates: "retention.ms:604800000;segment.bytes:52428800"
      version-control: "retention.ms:604800000;segment.bytes:52428800"
      edge: "retention.ms:604800000;segment.bytes:52428800"
      edge-event: "retention.ms:604800000;segment.bytes:52428800"
      housekeeper: "retention.ms:604800000;segment.bytes:52428800"
      housekeeper-reprocessing: "retention.ms:7776000000;segment.bytes:52428800"
      calculated-field: "retention.ms:604800000;segment.bytes:52428800"
      calculated-field-state: "retention.ms:604800000;segment.bytes:52428800"
      edqs-events: "retention.ms:86400000;segment.bytes:52428800"
      edqs-state: "retention.ms:-1;cleanup.policy:compact"
      edqs-requests: "retention.ms:180000;retention.bytes:1073741824"
      tasks: "retention.ms:604800000;segment.bytes:52428800"
```

### Default Topic Configurations

| Topic Type | Retention | Segment Size | Notes |
|------------|-----------|--------------|-------|
| Core | 7 days | 50 MB | General platform messages |
| Rule Engine | 7 days | 50 MB | Rule chain processing |
| Transport API | 7 days | 50 MB | Device API messages |
| JS Executor | 1 day | 50 MB | Script execution |
| Housekeeper | 7 days | 50 MB | Cleanup tasks |
| Housekeeper Reprocessing | 90 days | 50 MB | Failed task retry |
| EDQS Events | 1 day | 50 MB | Entity changes |
| EDQS State | Infinite | Compacted | State snapshots |
| Tasks | 7 days | 50 MB | Background jobs |

## Consumer Statistics Service

### Enable Consumer Monitoring

```yaml
queue:
  kafka:
    consumer-stats:
      enabled: true
      print-interval-ms: 60000
      kafka-response-timeout-ms: 1000
```

### Statistics Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.consumer-stats.enabled` | true | Enable consumer lag tracking |
| `queue.kafka.consumer-stats.print-interval-ms` | 60000 | Log interval (60 seconds) |
| `queue.kafka.consumer-stats.kafka-response-timeout-ms` | 1000 | Stats query timeout |

### Statistics Output

The consumer statistics service logs partition lag for all registered consumer groups:

```
[groupId] Topic partitions with lag: [
  [topic=tb_rule_engine, partition=0, committedOffset=1000, endOffset=1050, lag=50],
  [topic=tb_rule_engine, partition=1, committedOffset=2000, endOffset=2100, lag=100]
]
```

### Monitoring Metrics

```mermaid
graph TB
    subgraph "Consumer Statistics"
        CS[Consumer Stats Service]
        REG[Registered Consumer Groups]
        POLL[Periodic Polling]
    end

    subgraph "Metrics Collected"
        CO[Committed Offset]
        EO[End Offset]
        LAG[Partition Lag]
    end

    CS --> REG
    REG --> POLL
    POLL --> CO
    POLL --> EO
    CO --> LAG
    EO --> LAG
```

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Committed Offset | Last committed consumer offset | N/A |
| End Offset | Latest message offset in partition | N/A |
| Lag | End offset - Committed offset | > 10000 |

## Topic Cache Configuration

### Cache Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `queue.kafka.topics_cache_ttl_ms` | 300000 | Topic cache TTL (5 minutes) |

The topic cache reduces broker queries by caching topic existence checks.

## Admin Operations

### Topic Management

The Kafka admin client handles:

- Topic creation with configured replication and partitions
- Topic deletion
- Consumer group offset synchronization
- Consumer lag calculation

### Timeout Configuration

| Operation | Timeout | Description |
|-----------|---------|-------------|
| Topic creation | request.timeout.ms | Create new topic |
| Topic deletion | request.timeout.ms | Delete topic |
| Consumer offset fetch | kafka-response-timeout-ms | Stats queries |
| List topics | request.timeout.ms | Topic enumeration |

## Message Serialization

### Default Serializers

| Component | Serializer |
|-----------|------------|
| Key | StringSerializer |
| Value | ByteArraySerializer |

### Custom Headers

When debug logging is enabled, producers inject tracing headers:

| Header | Description |
|--------|-------------|
| `_producerId` | Client ID of producer |
| `_threadName` | Thread name (debug) |
| `_stackTrace*` | Call stack (trace level) |

## Performance Tuning

### High Throughput Configuration

```yaml
queue:
  kafka:
    batch.size: 65536
    linger.ms: 5
    compression.type: lz4
    max_poll_records: 16384
    buffer.memory: 67108864
```

### Low Latency Configuration

```yaml
queue:
  kafka:
    batch.size: 16384
    linger.ms: 0
    compression.type: none
    acks: 1
```

### High Reliability Configuration

```yaml
queue:
  kafka:
    acks: all
    retries: 3
    replication_factor: 3
    min.insync.replicas: 2
```

## Additional Properties

### Inline Properties

For properties not explicitly supported, use inline configuration:

```yaml
queue:
  kafka:
    other-inline: "property1:value1;property2:value2"
```

### Common Additional Properties

| Property | Description |
|----------|-------------|
| `min.insync.replicas` | Minimum replicas for acks=all |
| `unclean.leader.election.enable` | Allow unclean leader election |
| `message.max.bytes` | Maximum message size on broker |

## Troubleshooting

### Connection Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| Connection refused | Broker not running | Start Kafka brokers |
| SSL handshake failed | Certificate mismatch | Verify SSL configuration |
| Authentication failed | Invalid credentials | Check SASL configuration |

### Performance Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| High producer latency | Low batch.size | Increase batch.size |
| Consumer lag | Slow processing | Add consumers, increase partitions |
| Out of memory | Large buffer.memory | Reduce buffer.memory or add heap |

### Consumer Group Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| Rebalancing storm | Frequent consumer restarts | Stabilize services |
| Stuck consumer | max.poll.interval exceeded | Increase timeout or optimize processing |

## Common Pitfalls

### Connection and Bootstrap

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Single bootstrap server configured | Cluster outage causes complete connection failure | Use comma-separated list with multiple brokers: `bootstrap.servers: "kafka1:9092,kafka2:9092,kafka3:9092"` |
| Incorrect port number | "Connection refused" errors, services fail to start | Verify broker listener port configuration (default 9092 for plaintext, 9093 for TLS). Check broker `listeners` setting |
| DNS resolution failure for broker hostnames | Intermittent connectivity, timeout errors during DNS outages | Use IP addresses in `bootstrap.servers` or verify DNS reliability. Check `/etc/hosts` for local resolution |
| Network firewall blocking Kafka port | Silent connection timeout after 30s, no error details | Verify port 9092/9093 open with `telnet kafka-host 9092`. Check security groups, iptables, cloud firewall rules |
| Bootstrap servers specified with protocol prefix | "Unknown host" error from Kafka client | Remove `kafka://` prefix. Use bare `hostname:port` format: `kafka1:9092` not `kafka://kafka1:9092` |
| Private network Kafka with public ThingsBoard instance | "Broker metadata unreachable" error, can connect but can't produce/consume | Kafka advertised.listeners must be reachable from ThingsBoard. Use VPN/VPC peering or expose Kafka via load balancer with correct advertised addresses |

### Producer Configuration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Using acks=1 (default in some configs) | Data loss if broker leader fails before replication completes | Set `acks: all` for critical topics (TB_CORE, TB_RULE_ENGINE, TB_TRANSPORT). Verify `min.insync.replicas >= 2` |
| No compression enabled | Network bandwidth usage 5x higher, inter-datacenter costs increase | Enable `compression.type: lz4` (best throughput) or `snappy` (balanced). Avoid gzip (high CPU) unless bandwidth critical |
| Tiny batch size (default 16 KB) | High CPU overhead from frequent small sends, poor throughput (< 10K msg/sec) | Increase `batch.size: 65536` (64 KB) and `linger.ms: 5` for high-throughput topics. Monitor batching metrics |
| retries=0 for critical messages | Transient network failures cause permanent message loss | Set `retries: 10` with `retry.backoff.ms: 100`. Enable `enable.idempotence: true` to prevent duplicates (Kafka 0.11+, default 3.0+) |
| Message size exceeds broker limit | "RecordTooLargeException", messages dropped silently | Increase broker `message.max.bytes: 10485760` (10 MB) and producer `max.request.size`. Or split large payloads into chunks |
| Missing idempotence configuration | Duplicate messages delivered on retry, breaks exactly-once semantics | Enable `enable.idempotence: true`. Required for transactional writes. Enabled by default in Kafka 3.0+ |
| Buffer memory too small for burst traffic | Producer blocks on `send()`, application threads stall | Increase `buffer.memory: 67108864` (64 MB default may be insufficient). Monitor `buffer-available-bytes` metric |
| Request timeout too short for slow brokers | "Timeout expired" errors on legitimate requests during high load | Increase `request.timeout.ms: 60000` (60 seconds). Consider broker performance tuning if persistent |

### Consumer Configuration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| max.poll.interval.ms too short for slow processing | Consumer kicked from group during rule chain execution, constant rebalancing | Increase `max.poll.interval.ms: 600000` (10 minutes) for TB_RULE_ENGINE. Monitor processing time per batch |
| fetch.min.bytes too high for low-traffic topics | Latency spikes (seconds to minutes) waiting for enough data to accumulate | Reduce `fetch.min.bytes: 1024` (1 KB) for real-time topics. Use default only for high-throughput batch processing |
| max.poll.records too high for slow processing | Processing timeout despite messages available, consumer seems stuck | Reduce `max.poll.records: 100` for complex rule chains. Balance batch size vs processing time (target < max.poll.interval) |
| session.timeout.ms too short | False rebalances triggered by GC pauses, consumer constantly rejoining group | Increase `session.timeout.ms: 30000` (30 seconds). Tune JVM GC to avoid pauses > 10s |
| auto.offset.reset=latest for new consumer group | Messages published before consumer start are lost permanently | Use `auto_offset_reset: earliest` for TB_CORE, TB_RULE_ENGINE to process all messages. Use `latest` only for monitoring consumers |
| enable.auto.commit=true with manual processing | Message loss if consumer crashes between poll() and processing completion | Use `enable.auto.commit: false` with manual commit after successful processing. Leverage ThingsBoard processing strategies |
| Consumer group ID collision across environments | Production and staging/dev compete for same messages, both get partial data | Use unique group ID per environment: `tb-core-prod`, `tb-core-staging`, `tb-core-dev`. Include environment prefix in all group IDs |
| fetch.max.wait.ms too low (< 100ms) | Excessive broker polling, high CPU on brokers from frequent empty fetches | Increase `fetch.max.wait.ms: 500` (500ms). Balance latency vs broker load. Use lower values only for ultra-low-latency requirements |

### Topic Configuration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Single partition for high-throughput topic | No parallelism possible, single consumer bottleneck (< 50K msg/sec) | Set `partitions = consumer_count` (e.g., 10 partitions for TB_RULE_ENGINE). Calculate: `partitions = target_throughput / per_consumer_throughput` |
| Replication factor = 1 in production | Complete partition loss if single broker fails, no data recovery possible | Set `replication_factor: 3` minimum (recommend 3). Verify with `kafka-topics.sh --describe`. Requires >= 3 brokers |
| retention.ms too short (< 1 day) | Data loss during maintenance windows, incidents, or processing delays | Increase `retention.ms: 604800000` (7 days) for critical topics. Use 24 hours minimum for any production topic |
| min.insync.replicas = 1 with acks=all | Data loss despite acks=all if leader dies immediately after ack but before replication | Set `min.insync.replicas: 2` on broker/topic. Guarantees quorum write. Requires `replication_factor >= 3`, `acks: all` |
| Partition count mismatch with consumer count | Idle consumers (consumers > partitions) or overloaded consumers (partitions > consumers) | Match `partitions = max(consumer_count)` per service type. Plan for growth: use `partitions = 2x current_consumers` |
| segment.ms too large (days/weeks) | Delayed log cleanup, retention.ms not honored promptly, disk fills unexpectedly | Reduce `segment.ms: 3600000` (1 hour) for faster retention enforcement. Smaller segments = more files but quicker cleanup |

### Monitoring and Operations

| Pitfall | Impact | Solution |
|---------|--------|----------|
| No consumer lag monitoring | Processing delays go unnoticed for hours/days, SLA violations, data backlog crisis | Monitor `records-lag-max` metric per consumer group. Alert on lag > 10,000 messages. Use ThingsBoard consumer-stats or Kafka monitoring tools |
| Missing under-replicated partition alerts | Silent data loss risk, partitions with replication < target go unnoticed | Monitor broker `UnderReplicatedPartitions` metric. Alert immediately on > 0. Indicates broker failures or network issues |
| No disk space monitoring on brokers | Broker crashes when disk full, partitions offline, data loss | Monitor disk usage on all brokers. Alert at 80% capacity. Ensure `log.retention.bytes` set to prevent unbounded growth |
| Ignoring broker logs (ERROR/WARN) | Missed early warnings about replication failures, corruption, misconfigurations | Centralize broker logs (Splunk, ELK). Alert on ERROR/WARN patterns: "NotEnoughReplicas", "CorruptRecord", "LeaderNotAvailable" |
| No rebalance monitoring | Slow/stuck rebalances cause application downtime (minutes to hours), users blocked | Track `rebalance-latency-avg` consumer metric. Alert on rebalance > 30 seconds. Investigate partition count, consumer count, network |
| Consumer group lag divergence across partitions | Some consumers falling behind while others idle, uneven load distribution | Monitor per-partition lag, not just group total lag. Identify slow consumers, hot partitions. Consider partition key distribution |

### Performance and Tuning

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Default JVM heap for Kafka brokers (1 GB) | Frequent GC pauses, out of memory errors under load (> 10K msg/sec) | Increase broker heap: `-Xmx6g -Xms6g` minimum for production. Allocate 25% of RAM to JVM, rest for OS page cache |
| Untuned OS page cache settings | Poor read performance (10x slower), high disk I/O, consumer lag | Set `vm.swappiness=1`, increase `vm.dirty_ratio=80`. Linux page cache critical for Kafka performance - allocate 50%+ of RAM |
| Network buffer too small for high throughput | Packet drops under load, retransmissions, unstable throughput | Increase socket buffers: `net.core.rmem_max=134217728` (128 MB), `net.core.wmem_max=134217728`. Verify with `netstat -s` |
| Synchronous processing in consumer threads | Low throughput (< 1K msg/sec per consumer), poor CPU utilization | Use async processing with dedicated thread pools. Batch commit offsets. Leverage ThingsBoard's per-partition consumers with parallel processing |

## See Also

- [Queue Architecture](./queue-architecture.md) - Overall queue design
- [Partitioning](./partitioning.md) - Partition strategies
- [Processing Strategies](./processing-strategies.md) - Submit and retry strategies
