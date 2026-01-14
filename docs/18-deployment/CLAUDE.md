# CLAUDE.md - Deployment & Operations Section

This file provides guidance for working with the Deployment & Operations documentation section.

## Section Purpose

This section documents ThingsBoard deployment and operations:

- **Installation Options**: Docker, Kubernetes, bare metal, cloud platforms
- **Configuration**: Environment variables, service parameters
- **Monitoring & Operations**: Health checks, metrics, troubleshooting

## File Structure

```
18-deployment/
├── README.md                    # Deployment overview and architecture
├── installation-options.md      # Deployment methods, cloud platforms
├── configuration.md             # Environment variables, parameters
└── monitoring-operations.md     # Health checks, metrics, troubleshooting
```

## Writing Guidelines

### Audience

DevOps engineers and system administrators deploying ThingsBoard. Assume familiarity with containerization and infrastructure but not necessarily with ThingsBoard-specific patterns.

### Content Pattern

Deployment documents should include:

1. **Overview** - What the deployment option provides
2. **Prerequisites** - Requirements and dependencies
3. **Installation** - Step-by-step instructions
4. **Configuration** - Key settings
5. **Verification** - Health check procedures
6. **Pitfalls** - Common deployment issues
7. **See Also** - Related documentation

### Deployment Documentation Pattern

For deployment options:

```markdown
## Deployment Option

**Best For**: Use case summary

### Prerequisites

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| ... | ... | ... |

### Installation

\`\`\`bash
# Installation commands
\`\`\`

### Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| ... | ... | ... |

### Verification

\`\`\`bash
# Health check commands
\`\`\`

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Monolithic" for single-process deployment
- Use "Microservices" for distributed deployment
- Use "Cluster" for multi-node deployment
- Use "Node" for individual service instance
- Use "Health check" for service status verification

### Diagrams

Use Mermaid diagrams to show:

- Deployment architecture (`graph TB`)
- Cluster topology (`graph TB`)
- HA configuration (`graph TB`)
- Decision flowcharts (`graph TD`)

### Technology-Agnostic Rule

Focus on deployment patterns, not internal implementation:

**DO**: "ThingsBoard scales horizontally by adding more node instances behind a load balancer"
**DON'T**: "TbServiceDiscoveryService registers instances via ZkDiscoveryService.registerService()"

**DO**: "Health checks verify database, cache, and queue connectivity"
**DON'T**: "HealthCheckService calls DataSource.getConnection() and validates JDBC pool"

**DO**: "Configuration is provided via environment variables or YAML files"
**DON'T**: "TbYamlPropertySourceLoader extends PropertySourceLoader with SnakeYAML"

## Reference Sources

When updating this section, cross-reference:

- `ref/thingsboard.github.io-master/docs/user-guide/install/` - Installation guides
- `ref/thingsboard-master/application/src/main/resources/` - Configuration files
- `ref/thingsboard-master/msa/` - Microservice configurations

## Related Sections

- `01-architecture/` - System design
- `08-message-queue/` - Kafka configuration
- `11-microservices/` - Service types
- `09-security/` - Security configuration

## Common Tasks

### Documenting Installation Steps

1. List prerequisites clearly
2. Show exact commands to run
3. Include verification steps
4. Document common errors

### Documenting Configuration

1. Group related variables
2. Show default values
3. Explain valid ranges
4. Include examples

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/18-deployment/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **kubernetes-specialist** | `/kubernetes-specialist` | K8s deployments, Helm charts |
| **docker-expert** | `/docker-expert` | Container configuration, Compose |
| **devops-engineer** | `/devops-engineer` | CI/CD, monitoring, automation |
| **cloud-architect** | `/cloud-architect` | Multi-cloud deployments, AWS/Azure/GCP patterns |
| **grafana-expert** | `/grafana-expert` | Monitoring dashboards, Prometheus metrics |
| **ansible-expert** | `/ansible-expert` | Configuration management, automation playbooks |
| **technical-writer** | `/technical-writer` | Clear deployment documentation |

### When to Use Each Skill

- **Documenting Kubernetes deployment**: Use `/kubernetes-specialist` for K8s patterns
- **Documenting Docker setup**: Use `/docker-expert` for container configuration
- **Documenting cloud deployment**: Use `/cloud-architect` for cloud platform patterns
- **Explaining monitoring**: Use `/devops-engineer` for operational guidance
- **Documenting Grafana dashboards**: Use `/grafana-expert` for metrics visualization
- **Documenting Ansible playbooks**: Use `/ansible-expert` for configuration management
- **Writing installation guides**: Use `/technical-writer` for clarity

## Key Deployment Concepts

When documenting deployment, emphasize:

| Concept | Key Points |
|---------|------------|
| **Monolithic** | Single process, simple operations, limited scale |
| **Microservices** | Distributed, scalable, complex operations |
| **High Availability** | Multiple nodes, failover, redundancy |
| **Health Checks** | Readiness, liveness, startup probes |
| **Configuration** | Environment variables, YAML, ConfigMaps |
| **Backup/Recovery** | Database backups, disaster recovery |

## Deployment Options

| Option | Use Case | Scale |
|--------|----------|-------|
| Docker | Development, testing | < 1K devices |
| Docker Compose | Small deployments | < 10K devices |
| Kubernetes | Production at scale | 10K+ devices |
| Bare Metal | Maximum control | Any |

## Infrastructure Requirements

| Resource | Development | Production |
|----------|-------------|------------|
| CPU | 2 cores | 8+ cores |
| RAM | 4 GB | 16+ GB |
| Storage | 20 GB SSD | 100+ GB SSD |

## Common Pitfalls to Document

Ensure documentation covers these deployment issues:

| Pitfall | Description |
|---------|-------------|
| Resource underprovisioning | Insufficient CPU/RAM for workload |
| Missing database indexes | Slow queries on large datasets |
| Incorrect JVM settings | Memory issues with default heap |
| Queue partition count | Too few partitions limiting parallelism |
| SSL certificate issues | Expired or misconfigured certificates |
| Backup not tested | Recovery fails when needed |
| Upgrade without backup | Data loss during failed upgrade |
| Configuration drift | Inconsistent settings across nodes |

## Health Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/health` | Overall health status |
| `/ready` | Readiness probe |
| `/metrics` | Prometheus metrics |

## HA Configuration

| Component | HA Strategy |
|-----------|-------------|
| ThingsBoard | Multiple nodes behind LB |
| PostgreSQL | Primary-replica replication |
| Cassandra | Multi-node cluster |
| Kafka | Multi-broker cluster |
| Redis | Cluster or Sentinel |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
