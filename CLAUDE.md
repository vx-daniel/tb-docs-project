# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a ThingsBoard documentation project containing:
- **docs/** - Internal documentation (80 markdown files) derived from source code analysis
- **ref/thingsboard-master** - ThingsBoard platform source code (Java 17/Spring Boot 3.4.10 backend, Angular 18 frontend)
- **ref/thingsboard.github.io-master** - Official ThingsBoard documentation website (Jekyll)
- **.claude/skills/** - 38 specialized Claude Code skills for development workflows

## Documentation Structure

Documentation is organized in numbered sections for logical progression:

| Section | Topic | Files |
|---------|-------|-------|
| 01-architecture | System design, topology, multi-tenancy | 3 |
| 02-core-concepts | Entities, data models, identity | 9 |
| 03-actor-system | Concurrency, message-driven architecture | 4 |
| 04-rule-engine | Processing pipelines, node types | 5 |
| 05-transport-layer | Device protocols (MQTT, CoAP, HTTP, LWM2M, SNMP) | 7 |
| 06-api-layer | REST, WebSocket, authentication | 6 |
| 07-data-persistence | Cassandra, PostgreSQL, caching | 6 |
| 08-message-queue | Kafka configuration, partitioning | 4 |
| 09-security | Auth, authorization, tenant isolation | 3 |
| 10-frontend | Angular architecture, widget system | 2 |
| 11-microservices | Service types, EDQS, JS Executor | 7 |
| 12-edge | Edge computing, cloud sync, rule templates | 4 |
| 13-iot-gateway | Gateway architecture, protocol connectors | 3 |

`docs/GLOSSARY.md` contains comprehensive terminology with Mermaid diagrams showing concept relationships.

## ThingsBoard Reference Build Commands

**Backend (from ref/thingsboard-master):**
```bash
# Full build with Maven (skips tests)
./build.sh

# Build specific modules
./build.sh msa/web-ui,msa/tb-node

# Build protocol buffers
./build_proto.sh
```

**Frontend (from ref/thingsboard-master/ui-ngx):**
```bash
npm start                  # Dev server at localhost:4200
npm run build:prod         # Production build
npm run lint               # ESLint
npm run build:types        # Generate types
npm run build:icon-metadata # Generate icon metadata
```

## Technology Stack (ThingsBoard)

- **Backend**: Java 17, Spring Boot 3.4.10, Kafka 3.9.1, Cassandra/PostgreSQL
- **Frontend**: Angular 18.2.13, Angular Material, NgRx, Leaflet
- **Protocols**: MQTT, CoAP, HTTP, LWM2M, SNMP, WebSocket
- **Architecture**: Actor-based concurrency, microservices, multi-tenant

## Documentation Conventions

When writing or editing documentation in `docs/`:
- Use Mermaid diagrams for architecture visualization
- Each section has a README.md with contents table and "See Also" cross-references
- Lead with what to do, then explain why
- Use **bold** for UI elements, `backticks` for code/variables
- Keep headings action-oriented: "Set SAML before adding users" not "SAML configuration timing"

## Available Skills

Key skills available via `/skill-name` commands:

**IoT & Backend**: `iot-engineer`, `mqtt-expert`, `spring-boot-expert`, `grpc-expert`, `websocket-engineer`

**Documentation**: `docs-write`, `docs-review`, `technical-writer`, `configuration-reference-generator`

**Planning**: `create-plan`, `brainstorming`, `prd`, `compound-engineering`

**Research**: `web-research`, `research-analyst`

Use `/skill-name` to invoke any skill (e.g., `/docs-write` when creating documentation).
