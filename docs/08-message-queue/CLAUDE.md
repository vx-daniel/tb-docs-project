# CLAUDE.md - Message Queue Section

This file provides guidance for working with the Message Queue documentation section.

## Section Purpose

This section documents ThingsBoard's inter-service messaging:

- **Queue Architecture**: Topics, producers, consumers
- **Partitioning**: Hash-based distribution for ordering
- **Processing Strategies**: Submit and failure handling
- **Kafka Configuration**: Production deployment settings

## File Structure

```
08-message-queue/
├── README.md                 # Queue overview and selection guide
├── queue-architecture.md     # Topic structure and message routing
├── partitioning.md           # Hash-based distribution and rebalancing
├── processing-strategies.md  # Submit strategies and retry mechanisms
└── kafka-configuration.md    # Kafka-specific settings and tuning
```

## Writing Guidelines

### Audience

DevOps engineers and platform administrators configuring message queues. Assume familiarity with distributed systems but not necessarily with ThingsBoard's specific queue patterns.

### Content Pattern

Message queue documents should include:

1. **Overview** - What the component/concept does
2. **When to Use** - Selection criteria
3. **Configuration** - Settings with examples
4. **Trade-offs** - Pros/cons of different options
5. **Monitoring** - Key metrics to watch
6. **Pitfalls** - Common mistakes with solutions
7. **See Also** - Related documentation

### Queue Documentation Pattern

For queue concepts:

```markdown
## Concept Name

**Purpose**: What it does

### How It Works

Explanation with diagram

### Configuration

\`\`\`yaml
# Key settings
\`\`\`

### Trade-offs

| Option | Pros | Cons |
|--------|------|------|
| ... | ... | ... |

### When to Use

- Scenario 1
- Scenario 2

### Common Pitfalls

| Pitfall | Impact | Solution |
|---------|--------|----------|
| ... | ... | ... |
```

### Terminology

- Use "Queue" for the logical message channel (Main, HighPriority)
- Use "Topic" for the Kafka topic
- Use "Partition" for the distribution unit
- Use "Consumer Group" for coordinated consumers
- Use "Submit Strategy" for how messages enter processing
- Use "Processing Strategy" for how failures are handled

### Diagrams

Use Mermaid diagrams to show:

- Queue architecture (`graph TB`)
- Message flow (`sequenceDiagram`)
- Decision trees (`flowchart TD`)
- Failure scenarios (`graph LR`)

### Technology-Agnostic Rule

Focus on queue behavior, not implementation:

**DO**: "Messages are partitioned by entity ID to ensure ordering per device"
**DON'T**: "TbKafkaProducerTemplate uses MurmurHash3 on EntityId.toString()"

**DO**: "The RETRY_ALL strategy resubmits the entire message pack on any failure"
**DON'T**: "TbRuleEngineSubmitStrategy.RETRY_ALL invokes reprocess() on TbPackProcessingContext"

**DO**: "Queue consumers acknowledge messages after successful rule chain completion"
**DON'T**: "TbQueueConsumer.commit() calls KafkaConsumer.commitSync(offsets)"

## Strategy Comparison

When documenting, clarify trade-offs:

### Submit Strategies

| Strategy | Ordering | Throughput | Use Case |
|----------|----------|------------|----------|
| BURST | None | Highest | High-volume telemetry |
| SEQUENTIAL | Global | Lowest | Strict ordering |
| SEQUENTIAL_BY_ORIGINATOR | Per-entity | Medium | Counter updates |
| SEQUENTIAL_BY_TENANT | Per-tenant | Medium | Tenant isolation |

### Processing Strategies

| Strategy | Data Loss | Blocking Risk | Use Case |
|----------|-----------|---------------|----------|
| SKIP_ALL_FAILURES | Yes | None | Telemetry (loss ok) |
| RETRY_ALL | No | High | Must-not-lose |
| RETRY_FAILED_AND_TIMED_OUT | No | Medium | Balanced |

## Reference Sources

When updating this section, cross-reference:

- `ref/thingsboard.github.io-master/docs/user-guide/rule-engine-2-5/queues/` - Official queue docs
- `ref/thingsboard-master/common/queue/` - Queue abstractions
- `ref/thingsboard-master/rule-engine/rule-engine-components/` - Queue rule nodes

## Related Sections

- `04-rule-engine/` - Rule chains consume from queues
- `05-transport-layer/` - Transports produce to queues
- `11-microservices/` - Service-to-queue mapping
- `01-architecture/` - Overall system topology

## Common Tasks

### Documenting Queue Configuration

1. Show YAML configuration examples
2. Document environment variable alternatives
3. Include default values
4. Note version-specific features
5. Show Kafka-specific settings separately

### Documenting Processing Strategies

1. Explain when to use each strategy
2. Show failure scenarios with diagrams
3. Document retry behavior
4. Include troubleshooting steps

### Adding Kafka Tuning Recommendations

1. Base on production experience
2. Include partition count guidance
3. Document consumer settings
4. Show monitoring commands

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/08-message-queue/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **kafka-expert** | `/kafka-expert` | Kafka configuration, partitioning, consumer tuning |
| **microservices-architect** | `/microservices-architect` | Distributed messaging patterns |
| **technical-writer** | `/technical-writer` | Clear queue documentation |

### When to Use Each Skill

- **Documenting Kafka settings**: Use `/kafka-expert` for broker/consumer config
- **Explaining partitioning**: Use `/kafka-expert` for distribution patterns
- **Documenting queue architecture**: Use `/microservices-architect` for topology
- **Writing strategy guides**: Use `/technical-writer` for decision trees

## Key Queue Concepts

When documenting queues, emphasize:

| Concept | Key Points |
|---------|------------|
| **Service Decoupling** | Async communication via queues |
| **Hash Partitioning** | Entity ID determines partition |
| **Ordering Guarantees** | Only within same partition |
| **Acknowledgment** | After successful processing |
| **Consumer Groups** | Parallel processing with coordination |
| **Checkpoint Node** | Route to different queue mid-chain |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Queue blocking | RETRY strategies can block on failures |
| Message loss | SKIP strategies discard failed messages |
| Message amplification | RETRY_ALL reprocesses successful messages |
| Race conditions | BURST with state updates causes conflicts |
| Consumer lag | Too few partitions limits parallelism |
| Partition hotspots | Uneven entity distribution |

## Failure Scenario Documentation

For each processing strategy, document:

| Scenario | Content |
|----------|---------|
| **Single failure** | How one failed message affects pack |
| **External service down** | Behavior during outage |
| **Timeout** | How timeouts are handled |
| **Recovery** | Behavior after service recovery |

## Monitoring Documentation

For monitoring docs, ensure coverage of:

| Metric | Description |
|--------|-------------|
| **Consumer lag** | Messages behind head |
| **Processing rate** | Messages per second |
| **Failure rate** | Failed messages percentage |
| **Retry count** | Reprocessed messages |
| **Partition balance** | Distribution across partitions |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
