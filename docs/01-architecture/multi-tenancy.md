# Multi-Tenancy Architecture

## Overview

ThingsBoard implements a comprehensive multi-tenancy architecture that enables a single platform instance to serve multiple independent organizations (tenants) with complete data isolation, configurable resource limits, and flexible permission models. The architecture enforces tenant boundaries at every layer: database, API, messaging, and rule processing.

## Key Behaviors

1. **Complete Data Isolation**: Each tenant's data is logically separated; cross-tenant queries are not permitted.

2. **Hierarchical Organization**: Three-level hierarchy (Tenant → Customer → User) supports complex organizational structures.

3. **Profile-Based Configuration**: Tenant profiles define quotas, limits, and features without code changes.

4. **Usage Tracking**: Real-time monitoring of resource consumption with warning and disabling thresholds.

5. **Queue Isolation**: Optional dedicated message processing for high-priority tenants.

6. **Role-Based Access Control**: Fine-grained permissions at resource and entity levels.

## Tenant Hierarchy

### Organization Structure

```mermaid
graph TB
    subgraph "System Level"
        SYS[System Tenant]
        SA[System Admin]
    end

    subgraph "Tenant Level"
        T1[Tenant A]
        T2[Tenant B]
        TA1[Tenant Admin A]
        TA2[Tenant Admin B]
    end

    subgraph "Customer Level"
        C1[Customer 1]
        C2[Customer 2]
        C3[Customer 3]
    end

    subgraph "User Level"
        U1[User 1]
        U2[User 2]
        U3[User 3]
    end

    SYS --> T1
    SYS --> T2
    SA -.->|manages| T1
    SA -.->|manages| T2
    T1 --> TA1
    T2 --> TA2
    T1 --> C1
    T1 --> C2
    T2 --> C3
    C1 --> U1
    C2 --> U2
    C3 --> U3
```

### Entity Hierarchy

| Level | Entity | Scope | Key Relationships |
|-------|--------|-------|-------------------|
| System | System Tenant | Platform-wide | Owns default profiles, system settings |
| Tenant | Tenant | Organization | Owns devices, assets, rule chains, dashboards |
| Customer | Customer | Sub-organization | Assigned devices, assets, dashboards |
| User | User | Individual | Belongs to tenant and optionally customer |

### Authority Levels

```mermaid
graph LR
    subgraph "Authority Hierarchy"
        SYS_ADMIN[SYS_ADMIN]
        TENANT_ADMIN[TENANT_ADMIN]
        CUSTOMER_USER[CUSTOMER_USER]
    end

    subgraph "Access Scope"
        ALL[All Tenants]
        ONE[Single Tenant]
        CUST[Customer Data]
    end

    SYS_ADMIN --> ALL
    TENANT_ADMIN --> ONE
    CUSTOMER_USER --> CUST
```

| Authority | Description | Tenant Access | Customer Access |
|-----------|-------------|---------------|-----------------|
| SYS_ADMIN | Platform administrator | All tenants | Cannot access directly |
| TENANT_ADMIN | Organization administrator | Own tenant only | All within tenant |
| CUSTOMER_USER | End user | Own tenant only | Assigned customer only |

## Data Isolation

### Database-Level Isolation

Every entity implements a tenant identifier interface, ensuring tenant context is always present:

```mermaid
graph TB
    subgraph "Entity Interface"
        HT[HasTenantId Interface]
    end

    subgraph "Entities"
        DEV[Device]
        ASSET[Asset]
        DASH[Dashboard]
        RC[Rule Chain]
        ALARM[Alarm]
    end

    HT --> DEV
    HT --> ASSET
    HT --> DASH
    HT --> RC
    HT --> ALARM
```

All database tables include `tenant_id` as a core column, enabling:
- Index optimization for tenant-scoped queries
- Efficient data partitioning
- Clear ownership boundaries

### Query Isolation Pattern

```mermaid
sequenceDiagram
    participant User
    participant API as API Controller
    participant Auth as Auth Context
    participant DAO as Data Access
    participant DB as Database

    User->>API: Request devices
    API->>Auth: Get current tenant
    Auth-->>API: TenantId
    API->>DAO: findByTenantId(tenantId)
    DAO->>DB: SELECT * WHERE tenant_id = ?
    DB-->>DAO: Tenant's devices only
    DAO-->>API: Results
    API-->>User: Device list
```

### Isolation Layers

| Layer | Mechanism | Enforcement |
|-------|-----------|-------------|
| Database | tenant_id column | DAO query filters |
| API | SecurityUser context | Controller validation |
| Messaging | Tenant ID in message | Queue partitioning |
| Rule Engine | Per-tenant routing | Submission strategies |
| WebSocket | Session binding | Subscription filters |

## Tenant Profiles

### Profile Structure

```mermaid
graph TB
    subgraph "Tenant Profile"
        TP[TenantProfile]
        TPD[ProfileData]
        TPC[ProfileConfiguration]
        TPQ[QueueConfiguration]
    end

    TP --> TPD
    TPD --> TPC
    TPD --> TPQ

    subgraph "Configuration Categories"
        LIMITS[Entity Limits]
        RATES[Rate Limits]
        QUOTAS[Usage Quotas]
        FEATURES[Feature Flags]
    end

    TPC --> LIMITS
    TPC --> RATES
    TPC --> QUOTAS
    TPC --> FEATURES
```

### Profile Assignment

```mermaid
sequenceDiagram
    participant Admin as System Admin
    participant Profile as Tenant Profile
    participant Tenant as Tenant
    participant Limits as Limit Enforcement

    Admin->>Profile: Create profile with limits
    Admin->>Tenant: Assign profile
    Tenant->>Limits: Load configuration

    Note over Limits: All operations check<br/>against profile limits
```

### Entity Limits

| Limit | Description | Default |
|-------|-------------|---------|
| maxDevices | Maximum devices per tenant | Unlimited (0) |
| maxAssets | Maximum assets per tenant | Unlimited (0) |
| maxCustomers | Maximum customers per tenant | Unlimited (0) |
| maxUsers | Maximum users per tenant | Unlimited (0) |
| maxDashboards | Maximum dashboards | Unlimited (0) |
| maxRuleChains | Maximum rule chains | Unlimited (0) |
| maxEdges | Maximum edge gateways | Unlimited (0) |
| maxResourcesInBytes | Total resource storage | Unlimited (0) |
| maxResourceSize | Single resource file size | Unlimited (0) |

### Rate Limits

Rate limits use the format `"<count>:<window_seconds>"` with support for multiple windows:

```
"1000:1,20000:60" = 1000 per second AND 20000 per minute
```

| Limit | Scope | Description |
|-------|-------|-------------|
| transportTenantMsgRateLimit | Tenant-wide | Overall message rate |
| transportTenantTelemetryMsgRateLimit | Tenant-wide | Telemetry message rate |
| transportTenantTelemetryDataPointsRateLimit | Tenant-wide | Data point rate |
| transportDeviceMsgRateLimit | Per device | Device message rate |
| transportGatewayMsgRateLimit | Per gateway | Gateway message rate |
| tenantServerRestLimitsConfiguration | Tenant-wide | REST API rate |
| customerServerRestLimitsConfiguration | Per customer | Customer API rate |

### Usage Quotas

| Quota | Description | Tracking |
|-------|-------------|----------|
| maxTransportMessages | Total transport messages allowed | Monthly/custom period |
| maxTransportDataPoints | Total data points allowed | Monthly/custom period |
| maxREExecutions | Rule Engine execution count | Monthly/custom period |
| maxJSExecutions | JavaScript execution count | Monthly/custom period |
| maxTbelExecutions | TBEL execution count | Monthly/custom period |
| maxEmails | Email sending limit | Monthly/custom period |
| maxSms | SMS sending limit | Monthly/custom period |
| maxCreatedAlarms | Maximum alarms | Monthly/custom period |

### WebSocket Limits

| Limit | Description |
|-------|-------------|
| maxWsSessionsPerTenant | Concurrent sessions tenant-wide |
| maxWsSessionsPerCustomer | Sessions per customer |
| maxWsSessionsPerRegularUser | Sessions per user |
| maxWsSubscriptionsPerTenant | Concurrent subscriptions |

### Data Retention

| Setting | Description |
|---------|-------------|
| defaultStorageTtlDays | Default telemetry retention |
| alarmsTtlDays | Alarm retention period |
| rpcTtlDays | RPC message retention |
| queueStatsTtlDays | Queue statistics retention |
| ruleEngineExceptionsTtlDays | Exception log retention |
| maxDPStorageDays | Data point retention (0 = unlimited) |

## Usage Tracking

### Usage State Machine

```mermaid
stateDiagram-v2
    [*] --> ACTIVE: Usage below threshold
    ACTIVE --> WARNING: Usage > 80% of limit
    WARNING --> DISABLED: Usage >= 100% of limit
    WARNING --> ACTIVE: Usage drops below 80%
    DISABLED --> WARNING: Limit increased or usage reset
    DISABLED --> ACTIVE: Manual reset or new period
```

### Tracked Metrics

```mermaid
graph TB
    subgraph "Usage Categories"
        TRANSPORT[Transport State]
        STORAGE[Storage State]
        RE[Rule Engine State]
        JS[JavaScript State]
        EMAIL[Email State]
        SMS[SMS State]
        ALARM[Alarm State]
    end

    subgraph "States"
        ACTIVE[ACTIVE]
        WARNING[WARNING]
        DISABLED[DISABLED]
    end

    TRANSPORT --> ACTIVE
    TRANSPORT --> WARNING
    TRANSPORT --> DISABLED
```

| Metric Key | Description |
|------------|-------------|
| TRANSPORT_MSG_COUNT | Total transport messages |
| TRANSPORT_DP_COUNT | Total data points transported |
| STORAGE_DP_COUNT | Data points in storage |
| RE_EXEC_COUNT | Rule Engine executions |
| JS_EXEC_COUNT | JavaScript executions |
| TBEL_EXEC_COUNT | TBEL executions |
| EMAIL_EXEC_COUNT | Emails sent |
| SMS_EXEC_COUNT | SMS sent |
| CREATED_ALARMS_COUNT | Alarms created |
| ACTIVE_DEVICES | Count of active devices |
| INACTIVE_DEVICES | Count of inactive devices |

### Warning Threshold

The warning threshold (default 80%) triggers notifications before limits are reached:

```
Warning triggered when: current_usage > limit * warnThreshold
```

## Isolated Rule Engine

### Isolation Concept

```mermaid
graph TB
    subgraph "Standard Processing"
        Q1[Shared Queue]
        T1[Tenant A Messages]
        T2[Tenant B Messages]
        T3[Tenant C Messages]
    end

    subgraph "Isolated Processing"
        Q2[Dedicated Queue]
        T4[Tenant D Messages]
    end

    T1 --> Q1
    T2 --> Q1
    T3 --> Q1
    T4 --> Q2

    Q1 --> RE1[Rule Engine Pool]
    Q2 --> RE2[Dedicated Rule Engine]
```

### When to Use Isolation

| Scenario | Recommendation |
|----------|----------------|
| Standard SaaS deployment | Shared queues |
| High-volume tenant | Isolated queue |
| Strict SLA requirements | Isolated queue |
| Predictable latency needs | Isolated queue |
| Cost-sensitive deployment | Shared queues |

### Configuration

Enable isolation in tenant profile:

```yaml
isolatedTbRuleEngine: true
```

This creates:
- Dedicated message queue topics
- Separate consumer group
- Independent processing threads
- Per-tenant statistics tracking

## Queue Isolation

### Queue Configuration per Profile

```mermaid
graph TB
    subgraph "Tenant Profile"
        TP[Profile]
    end

    subgraph "Queue Configuration"
        Q1[Main Queue]
        Q2[HighPriority Queue]
        Q3[SequentialByOriginator Queue]
    end

    TP --> Q1
    TP --> Q2
    TP --> Q3

    subgraph "Queue Properties"
        TOPIC[Topic Name]
        PARTS[Partitions]
        STRAT[Submit Strategy]
        PROC[Processing Strategy]
    end

    Q1 --> TOPIC
    Q1 --> PARTS
    Q1 --> STRAT
    Q1 --> PROC
```

### Queue Configuration Options

| Setting | Description |
|---------|-------------|
| name | Queue identifier |
| topic | Underlying message broker topic |
| pollInterval | Consumer poll frequency (ms) |
| partitions | Number of queue partitions |
| consumerPerPartition | One consumer per partition |
| packProcessingTimeout | Message processing timeout |
| submitStrategy | How messages are submitted |
| processingStrategy | How messages are processed |

### Submission Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| SequentialByTenantId | Ordered by tenant | Tenant consistency |
| SequentialByOriginatorId | Ordered by entity | Entity consistency |
| Burst | High throughput | Best effort |

### Message Routing

```mermaid
sequenceDiagram
    participant Transport
    participant Router as Queue Router
    participant Profile as Tenant Profile
    participant Queue as Message Queue

    Transport->>Router: Message with TenantId
    Router->>Profile: Get queue config
    Profile-->>Router: Queue settings
    Router->>Queue: Route to appropriate queue

    Note over Queue: Partitioned by<br/>tenant or entity
```

## Access Control

### Permission Model

```mermaid
graph TB
    subgraph "Permission Check Flow"
        REQ[API Request]
        AUTH[Authentication]
        RES[Resource Check]
        ENT[Entity Check]
        RESULT[Allow/Deny]
    end

    REQ --> AUTH
    AUTH --> RES
    RES --> ENT
    ENT --> RESULT

    subgraph "Check Details"
        AUTH_D["Who is the user?<br/>What authority?"]
        RES_D["Can this role access<br/>this resource type?"]
        ENT_D["Does user's tenant<br/>own this entity?"]
    end

    AUTH --> AUTH_D
    RES --> RES_D
    ENT --> ENT_D
```

### Two-Tier Permission Checking

1. **Resource Level**: Can this user role access this type of resource?
2. **Entity Level**: Does the user's tenant/customer own this specific entity?

### Operations

| Operation | Description |
|-----------|-------------|
| CREATE | Create new entity |
| READ | Read entity data |
| WRITE | Update entity |
| DELETE | Remove entity |
| ASSIGN_TO_CUSTOMER | Assign to customer |
| UNASSIGN_FROM_CUSTOMER | Remove from customer |
| RPC_CALL | Execute remote procedure |
| READ_CREDENTIALS | View credentials |
| WRITE_CREDENTIALS | Modify credentials |
| READ_ATTRIBUTES | Read attributes |
| WRITE_ATTRIBUTES | Modify attributes |
| READ_TELEMETRY | Read time-series data |
| WRITE_TELEMETRY | Send telemetry |
| CLAIM_DEVICES | Claim unclaimed device |

### Protected Resources

| Category | Resources |
|----------|-----------|
| Core | DEVICE, ASSET, CUSTOMER, TENANT, USER, DASHBOARD |
| Configuration | DEVICE_PROFILE, ASSET_PROFILE, TENANT_PROFILE |
| Processing | RULE_CHAIN, ALARM, RPC |
| Advanced | ENTITY_VIEW, EDGE, WIDGETS_BUNDLE |
| System | ADMIN_SETTINGS, QUEUE, API_USAGE_STATE |

### Access Matrix

```mermaid
graph TB
    subgraph "SYS_ADMIN Access"
        SA_YES[All Tenants<br/>Tenant Profiles<br/>System Settings]
        SA_NO[Customer User Data<br/>Direct]
    end

    subgraph "TENANT_ADMIN Access"
        TA_YES[Own Tenant<br/>All Customers<br/>All Devices]
        TA_NO[Other Tenants<br/>Tenant Profiles]
    end

    subgraph "CUSTOMER_USER Access"
        CU_YES[Assigned Entities<br/>Own Dashboard Access]
        CU_NO[Other Customers<br/>Tenant Settings<br/>Rule Chains]
    end

    style SA_YES fill:#c8e6c9
    style SA_NO fill:#ffcdd2
    style TA_YES fill:#c8e6c9
    style TA_NO fill:#ffcdd2
    style CU_YES fill:#c8e6c9
    style CU_NO fill:#ffcdd2
```

| Resource | SYS_ADMIN | TENANT_ADMIN | CUSTOMER_USER |
|----------|-----------|--------------|---------------|
| Tenant | Full access | Own only | Denied |
| Tenant Profile | Full access | Denied | Denied |
| Customer | Denied | Own tenant | Own only |
| Device | Denied | Own tenant | Assigned only |
| Rule Chain | Denied | Own tenant | Denied |
| Dashboard | Denied | Own tenant | Assigned only |

## Cross-Tenant Operations

### System Administrator Capabilities

```mermaid
graph TB
    subgraph "System Admin Operations"
        CREATE_T[Create Tenant]
        MODIFY_T[Modify Tenant]
        DELETE_T[Delete Tenant]
        CREATE_P[Create Profile]
        ASSIGN_P[Assign Profile]
        VIEW_ALL[View All Tenants]
    end

    subgraph "Restrictions"
        NO_CUST[Cannot access<br/>customer data directly]
        NO_USER_OPS[Cannot perform<br/>tenant user operations]
    end

    SA[System Admin] --> CREATE_T
    SA --> MODIFY_T
    SA --> DELETE_T
    SA --> CREATE_P
    SA --> ASSIGN_P
    SA --> VIEW_ALL
    SA -.-> NO_CUST
    SA -.-> NO_USER_OPS
```

### System Tenant

The system tenant (UUID with all zeros) owns:
- Default tenant profiles
- System-wide settings
- Default device/asset profiles
- Widget bundles

### Tenant Listing

| Authority | Listing Scope |
|-----------|---------------|
| SYS_ADMIN | All tenants |
| TENANT_ADMIN | Own tenant only |
| CUSTOMER_USER | Not permitted |

## Security Validation Flow

### Complete Request Validation

```mermaid
sequenceDiagram
    participant Client
    participant Filter as Security Filter
    participant Validator as Access Validator
    participant Service as Entity Service
    participant DB as Database

    Client->>Filter: Request with JWT
    Filter->>Filter: Validate token
    Filter->>Filter: Extract SecurityUser

    alt Invalid Token
        Filter-->>Client: 401 Unauthorized
    end

    Filter->>Validator: Check resource access
    Validator->>Validator: Check authority level

    alt Authority denied
        Validator-->>Client: 403 Forbidden
    end

    Validator->>Service: Load entity
    Service->>DB: Query by ID
    DB-->>Service: Entity
    Service-->>Validator: Entity with tenantId

    Validator->>Validator: Verify tenant ownership

    alt Wrong tenant
        Validator-->>Client: 403 Forbidden
    end

    Validator-->>Client: 200 OK with data
```

### Validation Rules by Entity Type

| Entity | SYS_ADMIN | TENANT_ADMIN | CUSTOMER_USER |
|--------|-----------|--------------|---------------|
| Device | Denied | Tenant match | Customer assignment |
| Asset | Denied | Tenant match | Customer assignment |
| Customer | Denied | Tenant match | Own customer only |
| User | Denied | Tenant match | Own user only |
| Tenant | All allowed | Own only | Denied |
| TenantProfile | All allowed | Denied | Denied |

## Best Practices

### Tenant Profile Design

1. **Start with defaults**: Use unlimited (0) for initial profiles
2. **Monitor before limiting**: Understand usage patterns first
3. **Gradual limits**: Implement limits incrementally
4. **Warning thresholds**: Keep at 80% for early notification

### Data Isolation

1. **Always filter by tenant**: Never query without tenant context
2. **Validate ownership**: Check entity ownership before operations
3. **Use DAO abstractions**: Let the framework handle filtering

### Queue Configuration

1. **Default shared queues**: Start with shared for simplicity
2. **Isolate when needed**: Use isolation for specific SLA requirements
3. **Monitor queue depth**: Track per-tenant queue metrics

### Performance

1. **Index on tenant_id**: Ensure all tables have tenant indexes
2. **Partition large tables**: Consider tenant-based partitioning
3. **Cache profiles**: Profile lookups should be cached

## Configuration Example

### Tenant Profile Configuration

```yaml
name: "Standard Tier"
description: "Standard tenant limits"
isolatedTbRuleEngine: false
profileData:
  configuration:
    type: "DEFAULT"
    # Entity limits
    maxDevices: 1000
    maxAssets: 500
    maxCustomers: 50
    maxUsers: 100
    maxDashboards: 100
    maxRuleChains: 20

    # Rate limits
    transportTenantMsgRateLimit: "1000:1,30000:60"
    transportDeviceMsgRateLimit: "100:1"

    # Usage quotas
    maxTransportMessages: 10000000
    maxREExecutions: 5000000
    maxJSExecutions: 1000000

    # WebSocket limits
    maxWsSessionsPerTenant: 500
    maxWsSubscriptionsPerTenant: 1000

    # Retention
    defaultStorageTtlDays: 90
    alarmsTtlDays: 365

    # Thresholds
    warnThreshold: 0.8
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 403 Forbidden | Tenant mismatch | Verify entity ownership |
| Rate limit exceeded | Too many requests | Increase limits or throttle |
| Usage disabled | Quota exhausted | Reset or increase quota |
| Missing data | Wrong tenant context | Check authentication |
| Queue delays | Shared queue congestion | Consider isolation |

### Monitoring Points

- Per-tenant API usage
- Queue depth per tenant
- Rate limit rejections
- Usage state transitions
- Cross-tenant access attempts (should be zero)

## Implementation Details

### Tenant Profile Caching

**DefaultTbTenantProfileCache** implements dual-level caching:
- **Layer 1**: `Map<TenantProfileId, TenantProfile>` - profile by ID
- **Layer 2**: `Map<TenantId, TenantProfileId>` - tenant's assigned profile
- Uses `ReentrantLock` with double-checked locking pattern
- Cache invalidation broadcasts via `addListener()` callbacks

### Rate Limit Implementation

**DefaultRateLimitService** uses token bucket algorithm:
- **Cache**: Caffeine in-memory (TTL: 120 minutes, max 200k entries)
- **Key**: `RateLimitKey(api, level)` where level is TenantId or EntityId
- **Rate Limit Format**: `"capacity:duration_seconds,capacity:duration_seconds"` (e.g., `"1000:1,20000:60"`)
- **Enforcement**: `TbRateLimits.tryConsume()` returns true if within limit
- Triggers `RateLimitsTrigger` notification on limit exceeded

### API Usage State Tracking

**TenantApiUsageState** tracks resource consumption:
- Maintains `currentCycleValues` and `currentHourValues` (ConcurrentHashMap)
- Monthly cycle boundaries with hourly granularity
- State transitions via `checkStateUpdatedDueToThreshold()`:
  - `ENABLED`: usage < warnThreshold (default 80%)
  - `WARNING`: warnThreshold ≤ usage < threshold
  - `DISABLED`: usage ≥ threshold
- Uses `toMoreRestricted()` to combine multiple record keys per feature

**DefaultTbApiUsageStateService** processes usage messages:
- Per-entity lock prevents concurrent updates
- Counters: Additive accumulation across reports
- Gauges: Max per service, aggregated across instances
- Persists state changes to database and triggers notifications

### Access Control Implementation

**DefaultAccessControlService** routes permission checks by authority:

**Permission Check Flow:**
```
checkPermission(user, resource, operation)
  → getPermissionChecker(authority, resource)
  → PermissionChecker.hasPermission(user, operation, entityId, entity)
  → Returns boolean / throws ThingsboardException
```

**Authority-Specific Validation:**

| Authority | Tenant Check | Customer Check |
|-----------|--------------|----------------|
| SYS_ADMIN | Entity must be system-level (null tenant) | N/A |
| TENANT_ADMIN | `user.getTenantId() == entity.getTenantId()` | N/A |
| CUSTOMER_USER | Tenant match required | `user.getCustomerId() == entity.getCustomerId()` |

**Special Cases:**
- CLAIM_DEVICES: Bypasses customer check for customer users
- Dashboards: Uses `isAssignedToCustomer()` instead of direct ID comparison
- Public resources: Allow READ on null/NullUid tenant entities
- User self-modification: Customer users can modify own account

### Entity Limit Enforcement

**DefaultTenantProfileConfiguration** provides limit methods:
- `getEntitiesLimit(EntityType)`: Returns max for DEVICE, ASSET, CUSTOMER, etc.
- Entity creation services check limits before persisting
- Returns 0 for unlimited

### Profile Data Structure

**TenantProfile** entity uses lazy deserialization:
- `profileDataBytes`: Persisted JSON bytes
- `TenantProfileData`: Transient wrapper containing:
  - `TenantProfileConfiguration` (polymorphic interface)
  - `List<TenantProfileQueueConfiguration>` for queue definitions

**DefaultTenantProfileConfiguration** fields:
- Entity limits: `maxDevices`, `maxAssets`, `maxCustomers`, `maxUsers`, `maxDashboards`, `maxRuleChains`, `maxEdges`
- Usage thresholds: `maxTransportMessages`, `maxREExecutions`, `maxJSExecutions`, `maxEmails`, `maxSms`
- TTL settings: `defaultStorageTtlDays`, `alarmsTtlDays`, `rpcTtlDays`
- WebSocket limits: `maxWsSessionsPerTenant`, `maxWsSubscriptionsPerTenant`
- Warning threshold: `warnThreshold` (default 0.8)

## See Also

- [System Overview](./system-overview.md) - Platform architecture
- [Database Schema](../07-data-persistence/database-schema.md) - Tenant tables
- [Authentication](../06-api-layer/authentication.md) - JWT and security
- [Message Queue Architecture](../08-message-queue/queue-architecture.md) - Queue system
- [REST API Overview](../06-api-layer/rest-api-overview.md) - API structure
