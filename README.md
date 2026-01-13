# ThingsBoard Documentation Project

Internal technical documentation for the ThingsBoard IoT platform, derived from source code analysis. This documentation serves as a comprehensive guide for developers to understand, extend, and maintain ThingsBoard deployments.

## Overview

```
tb-docs-project/
├── docs/                    # 115 markdown documentation files
├── ref/
│   ├── thingsboard-master/  # ThingsBoard platform source code
│   └── thingsboard.github.io-master/  # Official documentation source
├── .claude/skills/          # 38 Claude Code skills for development
└── CLAUDE.md                # AI assistant instructions
```

## Documentation Structure

The documentation is organized into 18 sections for logical progression from architecture to deployment:

| # | Section | Description | Files |
|---|---------|-------------|-------|
| 01 | [Architecture](docs/01-architecture/) | System design, topology, multi-tenancy | 3 |
| 02 | [Core Concepts](docs/02-core-concepts/) | Entities, data models, identity, provisioning, OTA, profiles | 13 |
| 03 | [Actor System](docs/03-actor-system/) | Concurrency, message-driven architecture | 4 |
| 04 | [Rule Engine](docs/04-rule-engine/) | Processing pipelines, node types, TBEL, queues, analytics | 14 |
| 05 | [Transport Layer](docs/05-transport-layer/) | Device protocols (MQTT, CoAP, HTTP, LwM2M, SNMP) | 9 |
| 06 | [API Layer](docs/06-api-layer/) | REST, WebSocket, RPC, alarms, notifications | 9 |
| 07 | [Data Persistence](docs/07-data-persistence/) | PostgreSQL, Cassandra, TimescaleDB, caching | 8 |
| 08 | [Message Queue](docs/08-message-queue/) | Kafka configuration, partitioning | 4 |
| 09 | [Security](docs/09-security/) | Authentication, authorization, rate limiting | 4 |
| 10 | [Frontend](docs/10-frontend/) | Angular architecture, widget system | 2 |
| 11 | [Microservices](docs/11-microservices/) | Service types, EDQS, JS Executor | 7 |
| 12 | [Edge](docs/12-edge/) | Edge computing, cloud sync | 4 |
| 13 | [IoT Gateway](docs/13-iot-gateway/) | Gateway architecture, protocol connectors | 3 |
| 14 | [Integrations](docs/14-integrations/) | Cloud, LoRaWAN, messaging platforms | 4 |
| 15 | [TBMQ](docs/15-tbmq/) | MQTT broker architecture | 3 |
| 16 | [Trendz](docs/16-trendz/) | Analytics, visualizations, predictions | 4 |
| 17 | [Mobile App](docs/17-mobile-app/) | Flutter app, customization | 3 |
| 18 | [Deployment](docs/18-deployment/) | Installation, configuration, monitoring | 4 |

## Quick Start

### Reading the Documentation

Start with these key documents based on your role:

**For Platform Understanding:**
1. [Architecture Overview](docs/01-architecture/system-overview.md) - High-level system design
2. [Multi-Tenancy](docs/01-architecture/multi-tenancy.md) - Tenant isolation model
3. [Data Model](docs/02-core-concepts/data-model/) - Entity relationships

**For Device Development:**
1. [Device Provisioning](docs/02-core-concepts/device-provisioning.md) - How devices join
2. [Transport Layer](docs/05-transport-layer/) - MQTT, CoAP, HTTP protocols
3. [Device API](docs/06-api-layer/device-api.md) - Telemetry and RPC

**For Backend Development:**
1. [Rule Engine](docs/04-rule-engine/) - Data processing pipelines
2. [Actor System](docs/03-actor-system/) - Concurrency model
3. [Data Persistence](docs/07-data-persistence/) - Database configuration

**For Operations:**
1. [Deployment](docs/18-deployment/) - Installation guides
2. [Security](docs/09-security/) - Authentication and rate limiting
3. [Hybrid Storage](docs/07-data-persistence/hybrid-storage.md) - Scaling decisions

### Glossary

See [GLOSSARY.md](docs/GLOSSARY.md) for comprehensive terminology with Mermaid diagrams showing concept relationships.

## Technology Stack

### ThingsBoard Platform

| Component | Technology |
|-----------|------------|
| Backend | Java 17, Spring Boot 3.4.10 |
| Frontend | Angular 18.2.13, Angular Material |
| Message Queue | Kafka 3.9.1 |
| Databases | PostgreSQL, Cassandra, TimescaleDB |
| Protocols | MQTT, CoAP, HTTP, LwM2M, SNMP, WebSocket |
| Architecture | Actor-based concurrency, microservices |

### Reference Source Code

The `ref/` directory contains source code for analysis:

```bash
ref/thingsboard-master/       # Platform source (Java/Angular)
ref/thingsboard.github.io-master/  # Official docs (Jekyll)
```

**Building ThingsBoard (from ref/thingsboard-master):**

```bash
# Full backend build
./build.sh

# Frontend development
cd ui-ngx
npm start     # Dev server at localhost:4200
```

## Documentation Conventions

### Style Guidelines

- **Mermaid diagrams** for architecture visualization
- **Tables** for configuration options and comparisons
- **Code blocks** for configuration examples
- **Bold** for UI elements, `backticks` for code/variables
- Action-oriented headings: "Configure rate limiting" not "Rate limiting configuration"

### Cross-References

Each section README includes:
- Contents table with descriptions
- Key concepts summary
- See Also links to related sections

### Target Audience

This documentation targets **junior to mid-level developers** who need to:
- Understand ThingsBoard's architecture
- Extend platform functionality
- Deploy and operate ThingsBoard
- Integrate devices and external systems

## Contributing

### Adding Documentation

1. Place new files in the appropriate `docs/##-section/` directory
2. Update the section's `README.md` contents table
3. Add cross-references to related documents
4. Include Mermaid diagrams for complex concepts
5. Update `CLAUDE.md` file counts if adding new files

### Documentation Quality

- Lead with what to do, then explain why
- Keep explanations concise and actionable
- Use real configuration examples
- Include troubleshooting sections
- Avoid Java-specific implementation details (focus on behavior/contracts)

## Project Maintenance

### File Counts

Current documentation: **115 markdown files**

Update `CLAUDE.md` when adding/removing documentation files.

### Claude Code Skills

The `.claude/skills/` directory contains 38 specialized skills for:
- IoT development (`mqtt-expert`, `grpc-expert`)
- Documentation (`docs-write`, `docs-review`)
- Planning (`create-plan`, `brainstorming`)
- Research (`web-research`, `research-analyst`)

Invoke skills with `/skill-name` in Claude Code.

## License

This documentation project is for internal use. ThingsBoard is licensed under Apache 2.0.

## Resources

- [ThingsBoard Official Docs](https://thingsboard.io/docs/)
- [ThingsBoard GitHub](https://github.com/thingsboard/thingsboard)
- [ThingsBoard Community](https://github.com/thingsboard/thingsboard/discussions)


