# CLAUDE.md - TBMQ Section

This file provides guidance for working with the TBMQ documentation section.

## Section Purpose

This section documents TBMQ - ThingsBoard MQTT Broker:

- **TBMQ Architecture**: High-performance broker, clustering, scalability
- **MQTT Features**: Protocol support, QoS, sessions, shared subscriptions
- **Integration**: ThingsBoard platform connectivity

## File Structure

```
15-tbmq/
├── README.md              # TBMQ overview and capabilities
├── tbmq-architecture.md   # Broker architecture, clustering, scaling
└── mqtt-features.md       # Protocol support, QoS, sessions
```

## Writing Guidelines

### Audience

IoT architects and DevOps engineers deploying MQTT infrastructure. Assume familiarity with MQTT concepts but not necessarily with TBMQ-specific patterns.

### Content Pattern

TBMQ documents should include:

1. **Overview** - What the feature does
2. **Architecture** - How it scales and clusters
3. **Configuration** - Broker settings
4. **Performance** - Benchmarks and tuning
5. **Integration** - ThingsBoard connectivity
6. **Pitfalls** - Common deployment issues
7. **See Also** - Related documentation

### MQTT Feature Documentation Pattern

For MQTT features:

```markdown
## Feature Name

**MQTT Version**: 3.1 / 3.1.1 / 5.0

### Behavior

| Aspect | Description |
|--------|-------------|
| ... | ... |

### Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| ... | ... | ... |

### Performance Impact

| Scenario | Impact |
|----------|--------|
| ... | ... |

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "TBMQ" for the ThingsBoard MQTT Broker product
- Use "Broker" for the MQTT server component
- Use "Client" for MQTT publishers/subscribers
- Use "Session" for client connection state
- Use "Shared Subscription" for load-balanced subscriptions

### Diagrams

Use Mermaid diagrams to show:

- Broker architecture (`graph TB`)
- Cluster topology (`graph LR`)
- Message flow (`sequenceDiagram`)
- Client types (`graph TB`)

### Technology-Agnostic Rule

Focus on MQTT behavior, not implementation:

**DO**: "TBMQ supports shared subscriptions for load balancing across multiple consumers"
**DON'T**: "SharedSubscriptionProcessor uses ConsumerGroup with Kafka-backed distribution"

**DO**: "Device client sessions persist messages in Redis when the client is offline"
**DON'T**: "DeviceSessionManager calls RedisSessionStore.persistQoS1Messages()"

**DO**: "Application clients use Kafka-backed storage for durable message delivery"
**DON'T**: "ApplicationMsgProcessor extends AbstractMsgProcessor with KafkaTemplate"

## Reference Sources

When updating this section, cross-reference:

- TBMQ official documentation
- MQTT 3.1.1 and 5.0 specifications

## Related Sections

- `05-transport-layer/mqtt.md` - ThingsBoard MQTT transport
- `08-message-queue/` - Internal queue architecture
- `14-integrations/` - Platform integrations

## Common Tasks

### Documenting MQTT Protocol Features

1. Document MQTT version support
2. Explain QoS level behavior
3. Document session persistence
4. Include shared subscription patterns

### Documenting Cluster Configuration

1. Document node configuration
2. Show scaling patterns
3. Explain load balancing
4. Include HA considerations

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/15-tbmq/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **mqtt-expert** | `/mqtt-expert` | MQTT protocol, QoS, sessions |
| **kafka-expert** | `/kafka-expert` | Kafka-backed message storage |
| **technical-writer** | `/technical-writer` | Clear broker documentation |

### When to Use Each Skill

- **Documenting MQTT features**: Use `/mqtt-expert` for protocol details
- **Explaining storage architecture**: Use `/kafka-expert` for Kafka patterns
- **Writing deployment guides**: Use `/technical-writer` for clarity

## Key TBMQ Concepts

When documenting TBMQ, emphasize:

| Concept | Key Points |
|---------|------------|
| **Device Clients** | Optimized for many-to-one, Redis-backed sessions |
| **Application Clients** | Kafka-backed, consumer group load balancing |
| **Shared Subscriptions** | Load-balanced message distribution |
| **QoS Levels** | 0 (at most once), 1 (at least once), 2 (exactly once) |
| **Session Persistence** | Clean session vs persistent session behavior |
| **Clustering** | Horizontal scaling to 100M+ connections |

## Performance Benchmarks

| Metric | Single Node | Cluster |
|--------|-------------|---------|
| Connections | 4M+ | 100M+ |
| Throughput | 3M+ msg/sec | Linear scale |
| Latency | Sub-millisecond | Sub-millisecond |

## Common Pitfalls to Document

Ensure documentation covers these TBMQ issues:

| Pitfall | Description |
|---------|-------------|
| Client type mismatch | Using DEVICE type for backend services |
| Session not persistent | Losing messages due to clean session |
| QoS overhead | QoS 2 performance impact at scale |
| Shared subscription syntax | Incorrect topic filter format |
| Cluster rebalancing | Message ordering during node changes |
| Resource sizing | Underprovisioned Redis or Kafka |

## Authentication Documentation

For auth docs, ensure coverage of:

| Method | Description |
|--------|-------------|
| Basic | Username/password authentication |
| X.509 | Certificate-based authentication |
| JWT | Token-based authentication |
| SCRAM | MQTT 5.0 enhanced authentication |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
