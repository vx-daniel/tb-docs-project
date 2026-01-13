# Data Persistence

## Overview

The data persistence layer handles storage and retrieval of all platform data. ThingsBoard uses a flexible architecture that separates entity storage from time-series storage, allowing you to choose the right database backend for your scale and requirements.

## Quick Start: Choosing Your Storage Backend

```mermaid
flowchart TD
    START[How many messages<br/>per second?] --> Q1{< 1,000 msg/sec?}
    Q1 -->|Yes| PG[PostgreSQL Only<br/>Simplest setup]
    Q1 -->|No| Q2{Need horizontal<br/>scaling?}
    Q2 -->|No| TS[PostgreSQL + TimescaleDB<br/>Best compression]
    Q2 -->|Yes| CASS[PostgreSQL + Cassandra<br/>Unlimited scale]

    PG --> DONE1[Start here for<br/>development & small deployments]
    TS --> DONE2[Great for medium<br/>production workloads]
    CASS --> DONE3[Required for large<br/>enterprise deployments]

    style PG fill:#e8f5e9
    style TS fill:#e3f2fd
    style CASS fill:#fff3e0
```

| Backend | Best For | Complexity | Max Throughput |
|---------|----------|------------|----------------|
| PostgreSQL only | Development, < 1K devices | Low | ~2,000 msg/sec |
| PostgreSQL + TimescaleDB | Production, 1K-50K devices | Medium | ~5,000 msg/sec |
| PostgreSQL + Cassandra | Enterprise, 50K+ devices | High | 30,000+ msg/sec |

## Contents

| Document | Description |
|----------|-------------|
| [Hybrid Storage](./hybrid-storage.md) | When and how to use PostgreSQL + Cassandra together |
| [Database Schema](./database-schema.md) | PostgreSQL schema, tables, indexes, and relationships |
| [Time-Series Storage](./timeseries-storage.md) | Telemetry storage strategies, partitioning, and aggregation |
| [TimescaleDB Storage](./timescale-storage.md) | TimescaleDB extension configuration and optimization |
| [Cassandra Storage](./cassandra-storage.md) | NoSQL time-series backend configuration and optimization |
| [Attribute Storage](./attribute-storage.md) | Key-value attribute persistence and versioning |
| [Caching](./caching.md) | Redis and Caffeine caching layers and invalidation |

## Architecture Overview

### What Goes Where

```mermaid
graph TB
    subgraph "PostgreSQL (Always Required)"
        E1[Devices & Assets]
        E2[Users & Customers]
        E3[Rule Chains]
        E4[Dashboards]
        E5[Alarms]
        E6[Attributes]
    end

    subgraph "Time-Series Backend (Choose One)"
        TS1[(PostgreSQL)]
        TS2[(TimescaleDB)]
        TS3[(Cassandra)]
    end

    subgraph "Telemetry Data"
        T1[Temperature readings]
        T2[GPS coordinates]
        T3[Sensor values]
        T4[Device metrics]
    end

    T1 & T2 & T3 & T4 --> TS1
    T1 & T2 & T3 & T4 --> TS2
    T1 & T2 & T3 & T4 --> TS3

    style E1 fill:#e3f2fd
    style E2 fill:#e3f2fd
    style E3 fill:#e3f2fd
    style E4 fill:#e3f2fd
    style E5 fill:#e3f2fd
    style E6 fill:#e3f2fd
    style TS1 fill:#e8f5e9
    style TS2 fill:#e8f5e9
    style TS3 fill:#fff3e0
```

### Data Flow

```mermaid
sequenceDiagram
    participant Device
    participant Transport as Transport Layer
    participant RE as Rule Engine
    participant Cache as Cache Layer
    participant DB as Database

    Device->>Transport: Telemetry data
    Transport->>RE: Process message

    RE->>Cache: Check cache
    alt Cache hit
        Cache-->>RE: Cached data
    else Cache miss
        RE->>DB: Query database
        DB-->>RE: Data
        RE->>Cache: Update cache
    end

    RE->>DB: Save telemetry
    DB-->>RE: Success
```

## Key Concepts

### Dual Storage Pattern

ThingsBoard stores time-series data in two forms:

| Store | Purpose | Query Pattern |
|-------|---------|---------------|
| **Historical** | All data points over time | Range scans for charts |
| **Latest Values** | Most recent reading per key | Point lookups for dashboards |

This separation optimizes both real-time monitoring (fast latest value lookups) and historical analysis (efficient time-range queries).

### Partitioning

Time-series data is automatically partitioned by time:

```mermaid
graph LR
    subgraph "Monthly Partitions"
        P1[January 2024]
        P2[February 2024]
        P3[March 2024]
    end

    subgraph "Query for Feb 15"
        Q[Query] --> P2
    end

    subgraph "Benefits"
        B1[Skip irrelevant data]
        B2[Fast time-range queries]
        B3[Easy data retention]
    end
```

### Caching Layers

```mermaid
graph TB
    subgraph "Two-Tier Caching"
        L1[Caffeine<br/>In-memory, per-node]
        L2[Redis<br/>Distributed, shared]
    end

    APP[Application] --> L1
    L1 -->|Miss| L2
    L2 -->|Miss| DB[(Database)]

    style L1 fill:#e8f5e9
    style L2 fill:#e3f2fd
```

| Layer | Type | Use Case |
|-------|------|----------|
| Caffeine (L1) | In-memory | Single-node deployments |
| Redis (L2) | Distributed | Multi-node clusters |

## Configuration Quick Reference

### PostgreSQL Only

```yaml
database:
  ts:
    type: sql
  ts_latest:
    type: sql
```

### PostgreSQL + TimescaleDB

```yaml
database:
  ts:
    type: timescale
  ts_latest:
    type: timescale
```

### PostgreSQL + Cassandra (Hybrid)

```yaml
database:
  ts:
    type: cassandra
  ts_latest:
    type: cassandra

cassandra:
  url: "node1:9042,node2:9042,node3:9042"
  keyspace_name: thingsboard
```

## Performance Guidelines

### Write Throughput

| Configuration | Expected Throughput |
|---------------|---------------------|
| PostgreSQL (single) | ~2,000 writes/sec |
| TimescaleDB | ~5,000 writes/sec |
| Cassandra (3 nodes) | ~30,000 writes/sec |

### Memory Requirements

| Component | Recommended |
|-----------|-------------|
| PostgreSQL | 25% of RAM for shared_buffers |
| Caffeine cache | 1-4 GB heap allocation |
| Redis | Separate process, 2-8 GB |
| Cassandra | 8-16 GB heap per node |

### Scaling Strategy

```mermaid
graph LR
    SMALL[Small<br/>1 PostgreSQL] --> MEDIUM[Medium<br/>+ TimescaleDB]
    MEDIUM --> LARGE[Large<br/>+ 3-node Cassandra]
    LARGE --> XLARGE[Enterprise<br/>+ Cassandra cluster]
```

## Common Tasks

### Monitoring Storage Health

| What to Monitor | Why |
|-----------------|-----|
| Disk usage | Plan capacity |
| Query latency | Detect slowdowns |
| Cache hit rate | Verify caching |
| Connection pool | Detect exhaustion |

### Data Retention

Configure TTL to automatically expire old data:

```yaml
sql:
  ttl:
    ts:
      enabled: true
      ts_key_value_ttl: 2592000  # 30 days in seconds
```

### Backup Strategy

| Database | Backup Method |
|----------|---------------|
| PostgreSQL | pg_dump, pg_basebackup |
| Cassandra | nodetool snapshot |
| Redis | RDB snapshots, AOF |

## See Also

- [Data Model](../02-core-concepts/data-model/) - What data is stored
- [Message Queue](../08-message-queue/README.md) - Write path orchestration
- [Microservices](../11-microservices/README.md) - Service-to-database mapping
- [Deployment](../18-deployment/README.md) - Production setup guides
