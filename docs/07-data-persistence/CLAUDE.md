# CLAUDE.md - Data Persistence Section

This file provides guidance for working with the Data Persistence documentation section.

## Section Purpose

This section documents ThingsBoard's storage layer:

- **Database Backends**: PostgreSQL, TimescaleDB, Cassandra
- **Storage Patterns**: Entity storage, time-series, attributes
- **Caching**: Redis and Caffeine cache layers
- **Schema**: Table structures and relationships

## File Structure

```
07-data-persistence/
├── README.md              # Storage overview and backend selection
├── hybrid-storage.md      # PostgreSQL + Cassandra architecture
├── database-schema.md     # PostgreSQL schema and indexes
├── timeseries-storage.md  # Time-series storage strategies
├── timescale-storage.md   # TimescaleDB configuration
├── cassandra-storage.md   # Cassandra backend setup
├── attribute-storage.md   # Key-value attribute persistence
└── caching.md             # Redis and Caffeine caching
```

## Writing Guidelines

### Audience

DevOps engineers and DBAs setting up and optimizing ThingsBoard storage. Assume familiarity with database concepts but not necessarily with specific backends.

### Content Pattern

Data persistence documents should include:

1. **Overview** - What the storage component does
2. **When to Use** - Selection criteria and trade-offs
3. **Configuration** - Setup parameters with examples
4. **Schema/Structure** - Data organization
5. **Performance** - Tuning recommendations
6. **Monitoring** - Key metrics to watch
7. **Pitfalls** - Common mistakes
8. **See Also** - Related documentation

### Storage Documentation Pattern

For each storage backend:

```markdown
## Backend Name

**Best For**: Use case summary

### Prerequisites

- Version requirements
- Resource requirements

### Configuration

\`\`\`yaml
# Key configuration
\`\`\`

### Schema/Data Model

| Table/Column Family | Purpose |
|---------------------|---------|
| ... | ... |

### Performance Tuning

| Parameter | Recommended | Impact |
|-----------|-------------|--------|
| ... | ... | ... |

### Monitoring

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| ... | ... | ... |

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Time-series" for temporal data (telemetry)
- Use "Entity storage" for relational data (devices, users)
- Use "Partition" for data sharding units
- Use "TTL" for time-to-live data expiration
- Use "Hypertable" for TimescaleDB partitioned tables

### Diagrams

Use Mermaid diagrams to show:

- Storage architecture (`graph TB`)
- Data flow (`sequenceDiagram`)
- Decision trees (`flowchart TD`)
- Partitioning schemes (`graph LR`)

### Technology-Agnostic Rule

Focus on concepts and configuration, not implementation:

**DO**: "Time-series data is partitioned by month for efficient range queries"
**DON'T**: "TsKvPartitioner extends AbstractPartitioner using YearMonth boundaries"

**DO**: "The cache uses a two-tier architecture: local in-memory and distributed Redis"
**DON'T**: "CaffeineCacheManager wraps Caffeine<String, Object> with LoadingCache"

**DO**: "Cassandra stores telemetry using entity ID and timestamp as partition key"
**DON'T**: "CassandraTsKvDao uses PreparedStatement with TokenAware policy"

## Backend Comparison

When documenting, help readers choose:

| Aspect | PostgreSQL | TimescaleDB | Cassandra |
|--------|------------|-------------|-----------|
| **Complexity** | Low | Medium | High |
| **Scale** | <2K msg/s | <5K msg/s | 30K+ msg/s |
| **Compression** | Good | Excellent | Good |
| **Horizontal Scale** | No | Limited | Yes |
| **Best For** | Dev/Small | Medium prod | Enterprise |

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/user-guide/install/` - Installation guides
- `~/work/viaanix/thingsboard-master/dao/` - Data access implementations
- `~/work/viaanix/thingsboard-master/application/src/main/resources/` - Configuration files

## Related Sections

- `02-core-concepts/data-model/` - Data structures being stored
- `08-message-queue/` - Write path orchestration
- `11-microservices/` - Service-to-database mapping
- `18-deployment/` - Production deployment guides

## Common Tasks

### Documenting Configuration Changes

1. Verify against source in `~/work/viaanix/thingsboard-master/`
2. Show YAML configuration examples
3. Document environment variable alternatives
4. Include default values and valid ranges
5. Note version-specific features

### Documenting Schema Changes

1. Document table/column family structure
2. Explain partition key choices
3. Show index definitions
4. Document foreign key relationships
5. Include migration considerations

### Adding Performance Recommendations

1. Base recommendations on benchmarks
2. Include memory/CPU requirements
3. Document scaling thresholds
4. Provide monitoring queries/commands

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/07-data-persistence/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **postgres-expert** | `/postgres-expert` | PostgreSQL schema, indexing, query optimization |
| **cassandra-expert** | `/cassandra-expert` | Cassandra data modeling, partition keys, tuning |
| **timescaledb** | `/timescaledb` | TimescaleDB hypertables, compression, continuous aggregates |
| **database-administrator** | `/database-administrator` | HA configuration, backup/recovery, performance optimization |
| **technical-writer** | `/technical-writer` | Clear storage documentation |

### When to Use Each Skill

- **Documenting PostgreSQL schema**: Use `/postgres-expert` for indexing, queries
- **Documenting Cassandra storage**: Use `/cassandra-expert` for data modeling
- **Documenting TimescaleDB**: Use `/timescaledb` for hypertables, compression
- **Documenting HA and backups**: Use `/database-administrator` for operations
- **Explaining storage selection**: Use `/technical-writer` for decision guides
- **Performance tuning docs**: Use appropriate database skill

## Key Storage Concepts

When documenting storage, emphasize:

| Concept | Key Points |
|---------|------------|
| **Dual Storage** | Entities in PostgreSQL, time-series configurable |
| **Latest Values** | Separate store for fast current value lookups |
| **Partitioning** | Time-based partitioning for efficient queries |
| **TTL** | Automatic data expiration for retention |
| **Two-Tier Cache** | Caffeine (local) + Redis (distributed) |
| **Hybrid Mode** | PostgreSQL for entities, Cassandra for time-series |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Wrong backend for scale | PostgreSQL overwhelmed at high volume |
| Missing indexes | Slow queries on entity tables |
| Partition key hotspots | Cassandra performance issues |
| Cache invalidation | Stale data after direct DB updates |
| TTL misconfiguration | Data deleted too early or never |
| Connection pool exhaustion | Too many concurrent connections |

## Performance Documentation

For performance docs, ensure coverage of:

| Aspect | Content |
|--------|---------|
| **Write Throughput** | Messages per second by backend |
| **Query Latency** | Expected response times |
| **Memory Requirements** | Per-component recommendations |
| **Disk I/O** | Storage patterns and IOPS |
| **Scaling Triggers** | When to upgrade backend |

## Caching Documentation

For caching docs, ensure coverage of:

| Aspect | Content |
|--------|---------|
| **Cache Layers** | Caffeine (L1) vs Redis (L2) |
| **Cache Keys** | What data is cached |
| **TTL Settings** | Cache expiration times |
| **Invalidation** | When cache is cleared |
| **Hit Rate Monitoring** | How to measure effectiveness |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
