# Architecture

## Overview

This section covers the high-level system architecture of the ThingsBoard platform, including component organization, deployment topology, and multi-tenancy design. These documents provide the foundational understanding needed before diving into specific subsystems.

## Contents

| Document | Description |
|----------|-------------|
| [System Overview](./system-overview.md) | High-level architecture, technology stack, component responsibilities, and deployment topology |
| [Multi-Tenancy](./multi-tenancy.md) | Tenant isolation model, resource management, and hierarchical organization |

## Key Concepts

- **Microservices Architecture**: Platform is decomposed into specialized services (tb-node, transports, js-executor)
- **Actor-Based Concurrency**: Core processing uses actor model for scalability and isolation
- **Multi-Tenant Design**: Complete data and resource isolation between tenants
- **Horizontal Scalability**: All components designed for cluster deployment
- **Technology Stack**: Java 17, Spring Boot 3.4.10, Kafka, PostgreSQL/Cassandra

## See Also

- [Actor System](../03-actor-system/README.md) - Detailed concurrency model
- [Microservices](../11-microservices/README.md) - Individual service documentation
- [Message Queue](../08-message-queue/README.md) - Inter-service communication
