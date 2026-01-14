# CLAUDE.md - Architecture Section

This file provides guidance for working with the Architecture documentation section.

## Section Purpose

This section documents ThingsBoard's high-level system architecture:

- **System Overview**: Component organization, technology stack, deployment topology
- **Multi-Tenancy**: Tenant isolation model, resource management, hierarchical organization

## File Structure

```
01-architecture/
├── README.md              # Section overview and navigation
├── system-overview.md     # High-level architecture, tech stack, components
└── multi-tenancy.md       # Tenant isolation, resource management
```

## Writing Guidelines

### Audience

Architects and senior developers evaluating or onboarding to ThingsBoard. Provide the "big picture" before diving into details.

### Content Pattern

Architecture documents should include:

1. **Overview** - What the system/component does at a high level
2. **Key Components** - Major pieces and their responsibilities
3. **Interactions** - How components communicate
4. **Design Decisions** - Why things are built this way
5. **Trade-offs** - What was sacrificed for what benefit
6. **Diagrams** - Visual representation of structure and flow
7. **See Also** - Links to detailed implementation docs

### Terminology

- Use "tb-node" for the core ThingsBoard server component
- Use "Transport" for protocol-specific services (MQTT, HTTP, CoAP)
- Use "Actor" for the concurrency model entities
- Use "Tenant" for organization-level isolation
- Use "Partition" for data/load distribution units

### Diagrams

Use Mermaid diagrams to show:

- System topology (`graph TB` or `graph LR`)
- Component interactions (`sequenceDiagram`)
- Deployment architecture (`graph TB` with subgraphs)
- Data flow (`flowchart`)

### Technology-Agnostic Rule

Focus on architectural concepts, not implementation details:

**DO**: "ThingsBoard uses an actor-based concurrency model for scalable message processing"
**DON'T**: "TbActorSystem extends AbstractActorSystem using Akka-like mailbox patterns"

**DO**: "Services communicate asynchronously through a message queue"
**DON'T**: "TbQueueProducer sends TbProtoQueueMsg to Kafka using KafkaTemplate"

**DO**: "The platform supports horizontal scaling by partitioning workload across nodes"
**DON'T**: "PartitionService uses ConsistentHashPartitioner with MurmurHash3"

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/reference/` - Official architecture docs
- `~/work/viaanix/thingsboard-master/application/` - Core application structure
- `~/work/viaanix/thingsboard-master/common/` - Shared components
- `~/work/viaanix/thingsboard-master/msa/` - Microservices definitions

## Related Sections

- `02-core-concepts/` - Entities and data models
- `03-actor-system/` - Detailed concurrency model
- `08-message-queue/` - Inter-service communication
- `11-microservices/` - Individual service documentation
- `18-deployment/` - Installation and operations

## Common Tasks

### Updating System Overview

1. Verify changes against source in `~/work/viaanix/thingsboard-master/`
2. Ensure technology stack versions are current (check `pom.xml`, `package.json`)
3. Update component diagrams if responsibilities change
4. Cross-check with `11-microservices/` for consistency

### Documenting New Architectural Patterns

1. Create or update relevant document
2. Include rationale (why this pattern was chosen)
3. Document trade-offs explicitly
4. Add Mermaid diagram showing the pattern
5. Link to implementation details in other sections

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/01-architecture/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **software-architecture** | `/software-architecture` | SOLID principles, Clean Architecture, design patterns |
| **microservices-architect** | `/microservices-architect` | Service boundaries, communication patterns, distributed systems |
| **cloud-architect** | `/cloud-architect` | Multi-cloud strategies, scalable architectures, cost optimization |
| **technical-writer** | `/technical-writer` | Clear documentation structure and flow |

### When to Use Each Skill

- **Documenting system design decisions**: Use `/software-architecture` for patterns and principles
- **Explaining service decomposition**: Use `/microservices-architect` for distributed system concepts
- **Documenting cloud deployment**: Use `/cloud-architect` for multi-cloud patterns
- **Writing overview sections**: Use `/technical-writer` for clarity and structure
- **Documenting multi-tenancy**: Use `/microservices-architect` for isolation patterns

## Key Architectural Concepts

When documenting ThingsBoard architecture, emphasize:

| Concept | Description |
|---------|-------------|
| **Actor Model** | Message-driven concurrency with isolated state |
| **Multi-Tenancy** | Complete data and resource isolation |
| **Horizontal Scaling** | Stateless services, partitioned data |
| **Event-Driven** | Asynchronous communication via message queues |
| **Polyglot Persistence** | PostgreSQL + Cassandra/TimescaleDB |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
