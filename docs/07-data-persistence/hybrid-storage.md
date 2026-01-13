# Hybrid Storage Architecture

## Overview

ThingsBoard supports a hybrid storage architecture that separates **entity data** from **time-series data** into different database backends. This design allows you to optimize each storage type independently: PostgreSQL for relational entity data and Cassandra (or TimescaleDB) for high-volume time-series telemetry.

## Why Hybrid Storage?

### The Problem with Single-Database Deployments

In IoT applications, you have two fundamentally different data access patterns:

```mermaid
graph TB
    subgraph "Entity Data"
        E1[Devices]
        E2[Assets]
        E3[Customers]
        E4[Users]
    end

    subgraph "Time-Series Data"
        T1[Temperature readings]
        T2[GPS coordinates]
        T3[Sensor telemetry]
        T4[Device metrics]
    end

    subgraph "Access Patterns"
        E1 & E2 & E3 & E4 --> AP1[CRUD operations<br/>Complex queries<br/>Joins & relationships<br/>Transactions]
        T1 & T2 & T3 & T4 --> AP2[High-volume inserts<br/>Time-range queries<br/>Aggregations<br/>Data expiration]
    end
```

| Characteristic | Entity Data | Time-Series Data |
|----------------|-------------|------------------|
| Write pattern | Occasional updates | Constant streaming inserts |
| Read pattern | Random access, joins | Sequential time-range scans |
| Volume | Thousands to millions | Billions of data points |
| Schema | Complex relationships | Simple key-value-timestamp |
| Transactions | Required | Not needed |

### The Hybrid Solution

```mermaid
graph TB
    subgraph "ThingsBoard Server"
        RE[Rule Engine]
        API[REST API]
    end

    subgraph "Entity Storage"
        PG[(PostgreSQL)]
        PG --> |Devices, Assets| E1[Entity CRUD]
        PG --> |Users, Customers| E2[Authentication]
        PG --> |Rule Chains| E3[Configuration]
    end

    subgraph "Time-Series Storage"
        CASS[(Cassandra)]
        CASS --> |Telemetry| TS1[High-volume writes]
        CASS --> |Latest values| TS2[Real-time dashboards]
        CASS --> |Historical data| TS3[Analytics]
    end

    RE --> PG
    RE --> CASS
    API --> PG
    API --> CASS

    style PG fill:#e3f2fd
    style CASS fill:#fff3e0
```

## When to Use Hybrid Storage

### Decision Guide

```mermaid
flowchart TD
    START[Start] --> Q1{Expected message<br/>rate per second?}
    Q1 -->|< 1,000| SMALL[PostgreSQL Only]
    Q1 -->|1,000 - 10,000| MEDIUM{Need horizontal<br/>scaling?}
    Q1 -->|> 10,000| LARGE[Hybrid Mode<br/>Recommended]

    MEDIUM -->|No| TIMESCALE[PostgreSQL + TimescaleDB]
    MEDIUM -->|Yes| LARGE

    SMALL --> DONE1[Single PostgreSQL<br/>deployment]
    TIMESCALE --> DONE2[PostgreSQL with<br/>TimescaleDB extension]
    LARGE --> DONE3[PostgreSQL + Cassandra<br/>cluster]

    style DONE1 fill:#e8f5e9
    style DONE2 fill:#e3f2fd
    style DONE3 fill:#fff3e0
```

### Recommendations by Scale

| Scale | Devices | Messages/sec | Recommended Setup |
|-------|---------|--------------|-------------------|
| Small | < 1,000 | < 1,000 | PostgreSQL only |
| Medium | 1,000 - 10,000 | 1,000 - 5,000 | PostgreSQL + TimescaleDB |
| Large | 10,000 - 100,000 | 5,000 - 30,000 | PostgreSQL + Cassandra (3 nodes) |
| Enterprise | > 100,000 | > 30,000 | PostgreSQL + Cassandra cluster |

### Cost-Benefit Analysis

| Configuration | Pros | Cons |
|---------------|------|------|
| **PostgreSQL Only** | Simple setup, single backup, ACID transactions | Limited write throughput |
| **PostgreSQL + TimescaleDB** | Compression, automatic partitioning, SQL queries | Single node bottleneck |
| **PostgreSQL + Cassandra** | Unlimited horizontal scaling, high availability | More complex operations |

## Architecture Deep Dive

### Data Flow

```mermaid
sequenceDiagram
    participant Device
    participant Transport as Transport Layer
    participant RE as Rule Engine
    participant PG as PostgreSQL
    participant CASS as Cassandra

    Device->>Transport: Telemetry data
    Transport->>RE: Process message

    par Entity lookup
        RE->>PG: Get device by token
        PG-->>RE: Device entity
    and Save telemetry
        RE->>CASS: Save time-series data
        CASS-->>RE: Success
    end

    Note over RE: Update device activity
    RE->>PG: Update last_activity_ts
```

### Storage Responsibilities

```mermaid
graph TB
    subgraph "PostgreSQL Responsibilities"
        PG1[tenant - Multi-tenant isolation]
        PG2[device - Device registry]
        PG3[device_credentials - Authentication]
        PG4[rule_chain - Processing rules]
        PG5[dashboard - UI definitions]
        PG6[alarm - Alarm state]
        PG7[attribute_kv - Entity attributes]
    end

    subgraph "Cassandra Responsibilities"
        C1[ts_kv - Historical telemetry]
        C2[ts_kv_latest - Latest values]
        C3[ts_kv_partitions - Partition index]
    end

    style PG1 fill:#e3f2fd
    style PG2 fill:#e3f2fd
    style PG3 fill:#e3f2fd
    style PG4 fill:#e3f2fd
    style PG5 fill:#e3f2fd
    style PG6 fill:#e3f2fd
    style PG7 fill:#e3f2fd
    style C1 fill:#fff3e0
    style C2 fill:#fff3e0
    style C3 fill:#fff3e0
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_TS_TYPE` | `sql` | Time-series backend: `sql`, `timescale`, or `cassandra` |
| `DATABASE_TS_LATEST_TYPE` | `sql` | Latest values backend (can differ from ts.type) |
| `SPRING_DATASOURCE_URL` | - | PostgreSQL connection URL |
| `CASSANDRA_URL` | - | Cassandra contact points |

### PostgreSQL-Only Configuration

```yaml
# thingsboard.yml
database:
  ts:
    type: sql
  ts_latest:
    type: sql

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/thingsboard
    username: postgres
    password: postgres
```

### Hybrid Configuration (PostgreSQL + Cassandra)

```yaml
# thingsboard.yml
database:
  ts:
    type: cassandra
  ts_latest:
    type: cassandra

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/thingsboard
    username: postgres
    password: postgres

cassandra:
  url: "cassandra1:9042,cassandra2:9042,cassandra3:9042"
  keyspace_name: thingsboard
  local_datacenter: datacenter1
  query:
    ts_key_value_partitioning: MONTHS
```

### Split Configuration (Advanced)

You can use different backends for historical vs. latest values:

```yaml
# Historical in Cassandra, Latest in PostgreSQL
database:
  ts:
    type: cassandra      # High-volume historical writes
  ts_latest:
    type: sql            # Transactional latest values
```

This configuration is useful when:
- You need transactional consistency for latest values
- Dashboard queries primarily use latest values from PostgreSQL
- Historical data volume requires Cassandra's scale

## Performance Characteristics

### Write Performance

```mermaid
graph LR
    subgraph "Single PostgreSQL"
        PG_W[~2,000 writes/sec]
    end

    subgraph "TimescaleDB"
        TS_W[~5,000 writes/sec]
    end

    subgraph "Cassandra (3 nodes)"
        C_W[~30,000 writes/sec]
    end

    PG_W --> TS_W --> C_W

    style PG_W fill:#ffcdd2
    style TS_W fill:#fff9c4
    style C_W fill:#c8e6c9
```

### Query Performance Comparison

| Query Type | PostgreSQL | TimescaleDB | Cassandra |
|------------|------------|-------------|-----------|
| Point lookup | Fast | Fast | Very fast |
| Time-range scan | Moderate | Fast (compression) | Very fast |
| Aggregations | Fast (SQL) | Fast (built-in) | Slower (multi-partition) |
| Complex joins | Supported | Supported | Not supported |
| Full-text search | Supported | Supported | Limited |

### Cassandra Performance Benchmarks

Based on ThingsBoard performance testing:

| Configuration | Devices | Messages/sec | Hardware |
|---------------|---------|--------------|----------|
| Single node + Cassandra | 10,000 | 10,000 | c4.2xlarge (8 vCPU) |
| Single node + 3-node Cassandra | 20,000 | 30,000 | c4.2xlarge + 3x c4.xlarge |

Key optimizations that enable this performance:
1. **Asynchronous writes**: Non-blocking Cassandra driver API
2. **Connection pooling**: 32,768 max requests per connection
3. **Batched inserts**: Grouped writes to reduce round trips

## Migration Strategies

### PostgreSQL to Hybrid

```mermaid
flowchart TD
    A[Running on PostgreSQL] --> B[Deploy Cassandra cluster]
    B --> C[Configure hybrid mode]
    C --> D[Restart ThingsBoard]
    D --> E{Migrate historical data?}
    E -->|Yes| F[Run migration script]
    E -->|No| G[New data goes to Cassandra]
    F --> H[Verify migration]
    G --> H
    H --> I[Running in hybrid mode]
```

### Migration Steps

1. **Deploy Cassandra cluster** alongside existing PostgreSQL
2. **Update configuration** to use hybrid mode
3. **Restart ThingsBoard** - new telemetry writes go to Cassandra
4. **Optional**: Migrate historical data using export/import scripts
5. **Monitor** both databases during transition

### Data Migration Considerations

| Approach | Pros | Cons |
|----------|------|------|
| Fresh start | Simple, clean | Historical data lost |
| Parallel operation | No downtime | Temporary dual storage |
| Full migration | Complete history | Requires migration tooling |

## Operational Considerations

### Backup Strategy

```mermaid
graph TB
    subgraph "PostgreSQL Backup"
        PG_B[pg_dump / pg_basebackup]
        PG_B --> PG_S[(Entity data backup)]
    end

    subgraph "Cassandra Backup"
        C_B[nodetool snapshot]
        C_B --> C_S[(Time-series backup)]
    end

    subgraph "Recovery"
        PG_S --> REC[Restore to new cluster]
        C_S --> REC
    end
```

### Monitoring Points

| Component | Key Metrics |
|-----------|-------------|
| PostgreSQL | Connection pool usage, query latency, disk usage |
| Cassandra | Write latency, compaction pending, SSTable count |
| ThingsBoard | Message processing rate, queue depth |

### High Availability

```mermaid
graph TB
    subgraph "PostgreSQL HA"
        PG_M[Primary]
        PG_R[Replica]
        PG_M --> |Streaming replication| PG_R
    end

    subgraph "Cassandra HA"
        C1[Node 1]
        C2[Node 2]
        C3[Node 3]
        C1 --- C2
        C2 --- C3
        C3 --- C1
    end

    subgraph "ThingsBoard Cluster"
        TB1[Node 1]
        TB2[Node 2]
    end

    TB1 & TB2 --> PG_M
    TB1 & TB2 --> C1 & C2 & C3
```

## Best Practices

### For Small Deployments

1. Start with **PostgreSQL only** - simpler to manage
2. Monitor write throughput and latency
3. Upgrade to hybrid when approaching limits

### For Large Deployments

1. Use **3+ Cassandra nodes** for redundancy
2. Configure **QUORUM consistency** for reliability
3. Implement **proper backup procedures** for both databases
4. Set up **monitoring and alerting** for both systems

### Common Mistakes to Avoid

| Mistake | Impact | Solution |
|---------|--------|----------|
| Single Cassandra node | No fault tolerance | Use minimum 3 nodes |
| Shared PostgreSQL for entities and telemetry | Bottleneck | Use hybrid mode |
| Missing TTL configuration | Unbounded storage growth | Set appropriate TTL |
| Insufficient connection pooling | Connection timeouts | Increase pool size |

## Troubleshooting

### Connection Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| Cassandra connection timeout | Network/firewall | Check connectivity, ports 9042 |
| PostgreSQL pool exhausted | High concurrent load | Increase max connections |
| Slow writes | Wrong data center | Set correct `local_datacenter` |

### Performance Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| High write latency | Cassandra compaction | Add nodes or tune compaction |
| Query timeouts | Large time ranges | Use appropriate partition strategy |
| Memory pressure | Cache misconfiguration | Tune cache sizes |

## See Also

- [Database Schema](./database-schema.md) - Complete schema reference
- [Time-Series Storage](./timeseries-storage.md) - Detailed time-series patterns
- [Cassandra Storage](./cassandra-storage.md) - Cassandra configuration
- [Caching](./caching.md) - Cache layer optimization
