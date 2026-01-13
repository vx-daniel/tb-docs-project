# Time-Series Storage

## Overview

Time-series storage is the backbone of IoT data persistence in ThingsBoard. The platform implements a pluggable storage architecture that supports multiple database backends optimized for different scale and performance requirements. The design separates **historical data** (all data points over time) from **latest values** (most recent reading per key), enabling efficient queries for both real-time dashboards and historical analysis.

## Key Behaviors

1. **Dual Storage Pattern**: Historical data and latest values are stored separately for query optimization.

2. **Pluggable Backends**: Storage backend can be PostgreSQL, TimescaleDB, or Cassandra without changing application code.

3. **Time-Based Partitioning**: Data is partitioned by time intervals to manage growth and enable efficient cleanup.

4. **Key Dictionary**: String keys are mapped to integer IDs for storage efficiency.

5. **Asynchronous Operations**: All storage operations are non-blocking for high throughput.

6. **TTL Support**: Automatic data expiration based on configurable retention policies.

7. **Aggregation Engine**: Built-in support for MIN, MAX, AVG, SUM, COUNT across time windows.

## Storage Architecture

### Dual Storage Pattern

```mermaid
graph TB
    subgraph "Incoming Data"
        TS[Telemetry Service]
    end

    subgraph "Storage Layer"
        TS --> |"save()"| LATEST[Latest Values Store]
        TS --> |"save()"| HISTORY[Historical Store]
    end

    subgraph "Query Paths"
        LATEST --> |"findLatest()"| DASH[Real-time Dashboards]
        HISTORY --> |"findAll()"| CHARTS[Historical Charts]
        HISTORY --> |"aggregate()"| REPORTS[Analytics Reports]
    end

    style LATEST fill:#e1f5fe
    style HISTORY fill:#fff3e0
```

The dual storage pattern optimizes for two distinct access patterns:

| Store | Purpose | Query Pattern | Update Frequency |
|-------|---------|---------------|------------------|
| Latest Values | Current state | Point lookup by entity+key | Every data point |
| Historical | Time-series analysis | Range scans by time | Append-only |

### Storage Backend Options

```mermaid
graph TB
    subgraph "Application Layer"
        SVC[TimeseriesService]
    end

    subgraph "DAO Abstraction"
        DAO[TimeseriesDao Interface]
        LDAO[TimeseriesLatestDao Interface]
    end

    subgraph "Implementations"
        SQL[PostgreSQL DAO]
        SCALE[TimescaleDB DAO]
        CASS[Cassandra DAO]
    end

    SVC --> DAO
    SVC --> LDAO
    DAO --> SQL
    DAO --> SCALE
    DAO --> CASS
    LDAO --> SQL
    LDAO --> SCALE
    LDAO --> CASS
```

| Backend | Best For | Partitioning | Scaling Model |
|---------|----------|--------------|---------------|
| PostgreSQL | Small-medium deployments | Native table partitions | Vertical |
| TimescaleDB | Medium-large time-series | Automatic hypertable chunks | Vertical + compression |
| Cassandra | Large distributed deployments | Column family partitions | Horizontal |

### Hybrid Configuration

The platform supports using different backends for different storage types:

```mermaid
graph LR
    subgraph "Configuration"
        TS_TYPE["ts.type"]
        TS_LATEST["ts_latest.type"]
    end

    subgraph "Example: Hybrid Setup"
        TS_TYPE --> CASS_H[(Cassandra<br/>Historical)]
        TS_LATEST --> PG_L[(PostgreSQL<br/>Latest)]
    end
```

This enables optimizing each storage type independently - for example, using Cassandra for high-volume historical writes while keeping latest values in PostgreSQL for transactional consistency.

## Data Model

### Key Dictionary

String keys are normalized to integer IDs for storage efficiency:

```mermaid
sequenceDiagram
    participant App as Application
    participant Dict as Key Dictionary
    participant Store as Storage

    App->>Dict: "temperature"
    Dict-->>Dict: Lookup or create
    Dict-->>App: key_id = 42

    App->>Store: save(entity_id, 42, ts, value)
    Store-->>App: success
```

| Table | Column | Type | Description |
|-------|--------|------|-------------|
| ts_kv_dictionary | key | varchar(255) | Original key name |
| ts_kv_dictionary | key_id | serial | Auto-increment integer ID |

Benefits:
- Reduced storage footprint (4 bytes vs variable string length)
- Faster index comparisons
- Consistent key handling across backends

### Data Point Structure

Each time-series data point contains:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| entity_id | UUID | Yes | Entity the data belongs to |
| key | integer | Yes | Key ID from dictionary |
| ts | bigint | Yes | Timestamp in milliseconds |
| bool_v | boolean | No | Boolean value |
| str_v | varchar | No | String value |
| long_v | bigint | No | Integer value |
| dbl_v | double | No | Decimal value |
| json_v | jsonb | No | JSON object value |

Only one value column is populated per data point (mutually exclusive types).

### Latest Values Structure

Latest values include additional metadata for concurrency control:

| Field | Type | Description |
|-------|------|-------------|
| entity_id | UUID | Entity identifier |
| key | integer | Key ID |
| ts | bigint | Timestamp of latest value |
| *_v | various | Value columns (same as historical) |
| version | bigint | Optimistic lock version |

## Partitioning Strategies

### Time-Based Partitioning

Data is partitioned by time intervals to:
- Limit partition size for query performance
- Enable efficient TTL-based cleanup (drop entire partitions)
- Distribute writes across storage nodes

```mermaid
graph TB
    subgraph "Partition Strategy"
        direction LR
        P1[Jan 2024]
        P2[Feb 2024]
        P3[Mar 2024]
        P4[Apr 2024]
    end

    subgraph "Queries"
        Q1["Range: Jan 15 - Feb 15"]
        Q2["Range: Mar 1 - Mar 31"]
    end

    Q1 --> P1
    Q1 --> P2
    Q2 --> P3

    style P1 fill:#e8f5e9
    style P2 fill:#e8f5e9
    style P3 fill:#fff3e0
    style P4 fill:#f5f5f5
```

### Partition Intervals

| Interval | Duration | Use Case |
|----------|----------|----------|
| MINUTES | 1 minute | Very high frequency, short retention |
| HOURS | 1 hour | High frequency data |
| DAYS | 1 day | Standard IoT workloads |
| MONTHS | ~30 days | Default, most deployments |
| YEARS | ~365 days | Low volume, long retention |
| INDEFINITE | None | Single partition (simple, limited scale) |

### Backend-Specific Partitioning

**PostgreSQL**: Uses native declarative partitioning with range partitions on timestamp.

**TimescaleDB**: Uses hypertables with automatic chunk creation. Chunks are the equivalent of partitions but managed automatically.

**Cassandra**: Uses composite partition keys (entity_id + partition_ts) with partition timestamp calculated from the data timestamp.

### PostgreSQL Partition Management

Partitions are created automatically on first write:

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as Partition Cache
    participant Lock as ReentrantLock
    participant DB as PostgreSQL

    App->>Cache: Check partition exists
    alt Partition in cache
        Cache-->>App: Skip creation
    else Not in cache
        App->>Lock: Acquire lock
        Lock-->>App: Lock acquired
        App->>DB: CREATE TABLE ts_kv_YYYY_MM PARTITION OF ts_kv...
        alt Success
            DB-->>App: Created
        else Already exists
            DB-->>App: ConstraintViolation
        end
        App->>Cache: Add partition
        App->>Lock: Release lock
    end
```

**Partition naming**: `ts_kv_YYYY_MM` (for MONTHS) or `ts_kv_YYYY_MM_DD` (for DAYS)

**Double-checked locking**: Uses ReentrantLock to minimize contention during partition creation.

### Partition Tracking

For efficient queries, the system tracks which partitions exist:

```mermaid
graph LR
    subgraph "Query Optimization"
        Q[Query: entity X, last 7 days]
        PT[Partition Tracker]
        P1[Partition 1]
        P2[Partition 2]
        P3[Partition 3]
    end

    Q --> PT
    PT --> |"Exists"| P1
    PT --> |"Exists"| P2
    PT -.-> |"Skip empty"| P3

    style P3 fill:#f5f5f5,stroke-dasharray: 5 5
```

The partition tracker maintains a cache of existing partitions per entity, avoiding queries to empty partitions.

## Query Operations

### Query Types

```mermaid
graph TB
    subgraph "Query Operations"
        RAW[Raw Data Query]
        AGG[Aggregation Query]
        LATEST[Latest Value Query]
    end

    subgraph "Parameters"
        RAW --> |"key, startTs, endTs, limit, order"| R1[Exact data points]
        AGG --> |"key, startTs, endTs, interval, aggregation"| R2[Bucketed aggregates]
        LATEST --> |"entity, keys"| R3[Current values]
    end
```

### Raw Data Queries

Retrieve exact data points within a time range:

| Parameter | Type | Description |
|-----------|------|-------------|
| key | string | Time-series key name |
| startTs | long | Range start (inclusive) |
| endTs | long | Range end (exclusive) |
| limit | int | Maximum points to return |
| orderBy | enum | ASC or DESC by timestamp |

### Aggregation Queries

Compute statistics over time windows:

| Parameter | Type | Description |
|-----------|------|-------------|
| key | string | Time-series key name |
| startTs | long | Range start |
| endTs | long | Range end |
| interval | long | Bucket size |
| intervalType | enum | MILLISECONDS, HOURS, DAYS, MONTHS, YEARS |
| aggregation | enum | MIN, MAX, AVG, SUM, COUNT |
| timezone | string | For calendar-aligned buckets |

### Aggregation Functions

```mermaid
graph LR
    subgraph "Raw Data"
        D1[10]
        D2[20]
        D3[15]
        D4[25]
        D5[30]
    end

    subgraph "Aggregations"
        MIN["MIN = 10"]
        MAX["MAX = 30"]
        AVG["AVG = 20"]
        SUM["SUM = 100"]
        COUNT["COUNT = 5"]
    end

    D1 & D2 & D3 & D4 & D5 --> MIN
    D1 & D2 & D3 & D4 & D5 --> MAX
    D1 & D2 & D3 & D4 & D5 --> AVG
    D1 & D2 & D3 & D4 & D5 --> SUM
    D1 & D2 & D3 & D4 & D5 --> COUNT
```

### Query Flow

```mermaid
sequenceDiagram
    participant Client
    participant Service as TimeseriesService
    participant DAO as TimeseriesDao
    participant DB as Database

    Client->>Service: findAll(query)
    Service->>Service: Validate query
    Service->>Service: Resolve key IDs

    loop For each partition in range
        Service->>DAO: queryPartition(params)
        DAO->>DB: Execute query
        DB-->>DAO: Results
        DAO-->>Service: Partial results
    end

    Service->>Service: Merge & sort results
    Service-->>Client: TsKvEntry list
```

## Data Retention (TTL)

### TTL Mechanisms

```mermaid
graph TB
    subgraph "TTL Configuration"
        SYS[System-level TTL]
        WRITE[Per-write TTL]
    end

    subgraph "Enforcement"
        SYS --> CLEANUP[Cleanup Job]
        WRITE --> NATIVE[Native TTL<br/>Cassandra only]
        CLEANUP --> DROP[Drop old partitions]
        CLEANUP --> DELETE[Delete expired rows]
    end
```

### Configuration Options

| Setting | Default | Description |
|---------|---------|-------------|
| ts_key_value_ttl | 0 (disabled) | System-wide TTL in seconds |
| execution_interval_ms | 86400000 | Cleanup job frequency (1 day) |

### Cleanup Process

1. **Partition-based cleanup**: Drop entire partitions older than TTL
2. **Row-based cleanup**: Delete individual rows based on timestamp
3. **Latest value handling**: Optionally preserve or expire latest values

### TTL Best Practices

- Set TTL based on data analysis requirements
- Use partition-aligned TTL for efficient cleanup
- Consider different TTL for different data types (e.g., keep alarms longer than raw telemetry)

## Write Operations

### Write Flow

```mermaid
sequenceDiagram
    participant Producer as Data Producer
    participant Service as TimeseriesService
    participant Batch as Batch Processor
    participant Latest as Latest DAO
    participant History as Historical DAO

    Producer->>Service: save(entries, ttl)
    Service->>Batch: Queue for batching

    Note over Batch: Accumulate until<br/>batch size or timeout

    Batch->>Latest: saveBatch(latestValues)
    Batch->>History: saveBatch(historicalValues)

    Latest-->>Batch: Success
    History-->>Batch: Success
    Batch-->>Service: Complete
    Service-->>Producer: Success
```

### Batching Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `sql.ts.batch_size` | 10000 | Max entries per batch |
| `sql.ts.batch_max_delay` | 100 | Max wait before flush (ms) |
| `sql.ts.batch_threads` | 4 | Parallel batch processors |
| `sql.ts_latest.batch_size` | 1000 | Latest values batch size |
| `sql.ts_latest.batch_max_delay` | 50 | Latest values max delay (ms) |
| `sql.batch_sort` | true | Sort batches to prevent deadlocks |

### SQL Batch Queue Architecture

```mermaid
graph TB
    subgraph "Write Requests"
        W1[save(entity1)]
        W2[save(entity2)]
        W3[save(entity3)]
    end

    subgraph "TbSqlBlockingQueue"
        HASH[Hash Distribution]
        Q1[Queue 1]
        Q2[Queue 2]
        Q3[Queue N]
    end

    subgraph "Batch Processing"
        BP1[Batch Processor 1]
        BP2[Batch Processor 2]
        BP3[Batch Processor N]
    end

    W1 --> HASH
    W2 --> HASH
    W3 --> HASH
    HASH --> Q1
    HASH --> Q2
    HASH --> Q3
    Q1 --> BP1
    Q2 --> BP2
    Q3 --> BP3
    BP1 --> DB[(Database)]
    BP2 --> DB
    BP3 --> DB
```

### Hash-Based Queue Distribution

Writes are distributed across queues using entity ID hash:

```
queueIndex = (entityId.hashCode() & 0x7FFFFFFF) % batchThreads

Benefit: All writes for same entity go to same queue,
         maintaining per-entity ordering
```

### Batch Sorting

When `sql.batch_sort=true`, batches are sorted before execution:

| Sort Order | Purpose |
|------------|---------|
| entity_id | Group by entity |
| key | Group by metric |
| ts | Chronological order |

**Purpose**: Prevents deadlocks in clustered PostgreSQL by ensuring consistent lock acquisition order.

### Write Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| save() | Update both latest and historical | Normal telemetry |
| saveLatest() | Update latest only | Current state sync |
| saveWithoutLatest() | Update historical only | Backfill operations |

### Latest Value Deduplication

When multiple writes occur for the same entity+key within a batch:

```mermaid
graph LR
    subgraph "Incoming Writes"
        W1["temp=20 @ t1"]
        W2["temp=21 @ t2"]
        W3["temp=22 @ t3"]
    end

    subgraph "Batch Processing"
        DEDUP[Deduplication]
    end

    subgraph "Storage"
        LATEST["Latest: temp=22"]
        HIST["History: all 3 points"]
    end

    W1 & W2 & W3 --> DEDUP
    DEDUP --> |"Only newest"| LATEST
    DEDUP --> |"All points"| HIST
```

Only the newest value per entity+key is written to the latest store, reducing write amplification.

## Delete Operations

### Delete Modes

```mermaid
graph TB
    subgraph "Delete Options"
        DEL[Delete Request]
    end

    subgraph "Behaviors"
        DEL --> |"deleteLatest=false"| HIST_ONLY[Delete historical only]
        DEL --> |"deleteLatest=true"| BOTH[Delete both]
        DEL --> |"rewriteLatest=true"| REWRITE[Delete & recalculate latest]
    end
```

| Option | Description |
|--------|-------------|
| deleteLatest=false | Keep latest value, remove historical range |
| deleteLatest=true | Remove both latest and historical |
| rewriteLatestIfDeleted=true | After deleting, find next-most-recent value and set as latest |

### Latest Value Rewriting

When historical data is deleted and `rewriteLatestIfDeleted=true`:

```mermaid
sequenceDiagram
    participant Client
    participant Service
    participant LatestDAO
    participant HistoryDAO

    Client->>Service: delete(range, rewriteLatest=true)
    Service->>HistoryDAO: delete(range)
    Service->>HistoryDAO: findLatest(before deleted range)
    HistoryDAO-->>Service: Previous value
    Service->>LatestDAO: save(previous value)
    Service-->>Client: Success
```

## Performance Considerations

### Query Optimization

1. **Partition pruning**: Queries only scan partitions within the time range
2. **Key filtering**: Dictionary lookup happens once, then integer comparisons
3. **Index usage**: Primary key indexes on (entity_id, key, ts) enable efficient scans
4. **Result streaming**: Large result sets are streamed rather than loaded entirely

### Write Optimization

1. **Async batching**: Writes are accumulated and flushed in batches
2. **Parallel processing**: Multiple batch processors run concurrently
3. **Statement deduplication**: Redundant latest-value updates are eliminated
4. **Sort ordering**: SQL batches are sorted to prevent deadlocks in clusters

### Scaling Guidelines

| Scale | Backend | Configuration |
|-------|---------|---------------|
| < 1M points/day | PostgreSQL | Monthly partitions |
| 1M-100M points/day | TimescaleDB | Weekly chunks, compression |
| > 100M points/day | Cassandra | Distributed cluster, daily partitions |

### Memory Considerations

- **Partition cache**: Stores partition existence info (configurable size)
- **Key dictionary cache**: Maps string keys to IDs
- **Batch buffers**: Accumulate writes before flush

## Configuration Reference

### Database Type Selection

```yaml
database:
  ts:
    type: sql          # sql, timescale, or cassandra
  ts_latest:
    type: sql          # Can differ from ts.type
```

### PostgreSQL Settings

```yaml
sql:
  postgres:
    ts_key_value_partitioning: MONTHS    # DAYS, MONTHS, YEARS, INDEFINITE
  ts:
    batch_size: 10000
    batch_max_delay: 100
    batch_threads: 3
  ts_latest:
    batch_size: 1000
    batch_max_delay: 50
    update_by_latest_ts: true            # Only update if newer
  ttl:
    ts:
      enabled: true
      execution_interval_ms: 86400000    # 1 day
      ts_key_value_ttl: 0                # 0 = disabled
```

### TimescaleDB Settings

```yaml
sql:
  timescale:
    chunk_time_interval: 604800000       # 1 week in milliseconds
```

### Cassandra Settings

```yaml
cassandra:
  query:
    ts_key_value_partitioning: MONTHS
    ts_key_value_ttl: 0
    ts_key_value_partitions_max_cache_size: 100000
    default_fetch_size: 2000
```

### Query Limits

```yaml
database:
  ts:
    max_intervals: 700                   # Max aggregation buckets per query
```

## Entity View Integration

Entity Views can filter time-series access:

```mermaid
graph TB
    subgraph "Entity View"
        EV[Entity View]
        EV --> |"startTs"| START[Time Window Start]
        EV --> |"endTs"| END[Time Window End]
        EV --> |"keys"| KEYS[Allowed Keys]
    end

    subgraph "Query Processing"
        Q[Original Query]
        Q --> INTERSECT[Intersect with View]
        INTERSECT --> FILTERED[Filtered Query]
    end

    START --> INTERSECT
    END --> INTERSECT
    KEYS --> INTERSECT
```

Queries through an Entity View are automatically filtered to:
- Only allowed keys (whitelist)
- Only the configured time window
- The underlying entity's data

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Slow queries | Missing partition pruning | Ensure time range is specified |
| High memory usage | Large partition cache | Reduce cache size config |
| Write latency spikes | Batch size too large | Reduce batch_size |
| Missing latest values | TTL expired | Check TTL configuration |
| Query timeout | Too many intervals | Reduce time range or increase interval |

### Monitoring Points

- Batch queue depth
- Partition cache hit rate
- Query latency by type (raw vs aggregation)
- TTL cleanup job duration
- Write throughput (points/second)

## See Also

- [Database Schema](./database-schema.md) - Table definitions
- [Cassandra Storage](./cassandra-storage.md) - Cassandra-specific configuration
- [Caching](./caching.md) - Redis and Caffeine cache configuration
- [Telemetry Data Model](../02-core-concepts/data-model/telemetry.md) - Data structure
- [Message Queue Architecture](../08-message-queue/queue-architecture.md) - Data ingestion
- [Multi-Tenancy](../01-architecture/multi-tenancy.md) - Tenant isolation in storage
