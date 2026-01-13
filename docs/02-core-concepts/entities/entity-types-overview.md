# Entity Types Overview

## Overview

ThingsBoard uses a comprehensive entity type system to model all platform objects. Each entity type has a unique identifier format, database representation, and associated behaviors. As of ThingsBoard 4.3.0, there are 27+ distinct entity types.

This document helps you understand:
- What entity types exist
- How they relate to each other
- How they're organized and identified

---

## Entity Categories at a Glance

```mermaid
graph TB
    subgraph "Organization"
        ORG[Organization Entities]
        ORG --> TENANT[Tenant]
        ORG --> CUSTOMER[Customer]
        ORG --> USER[User]
    end

    subgraph "IoT"
        IOT[IoT Entities]
        IOT --> DEVICE[Device]
        IOT --> ASSET[Asset]
        IOT --> ALARM[Alarm]
    end

    subgraph "Configuration"
        CFG[Configuration Entities]
        CFG --> DP[Device Profile]
        CFG --> AP[Asset Profile]
        CFG --> TP[Tenant Profile]
    end

    subgraph "Processing"
        PROC[Processing Entities]
        PROC --> RC[Rule Chain]
        PROC --> RN[Rule Node]
    end

    subgraph "Visualization"
        VIZ[Visualization Entities]
        VIZ --> DASH[Dashboard]
        VIZ --> WB[Widget Bundle]
        VIZ --> WT[Widget Type]
    end

    style TENANT fill:#e3f2fd
    style DEVICE fill:#e8f5e9
    style DP fill:#fff3e0
    style RC fill:#f3e5f5
    style DASH fill:#fce4ec
```

---

## Entity Type Catalog

### Core Entities

The fundamental entities you'll work with most often.

```mermaid
graph LR
    subgraph "Core Entities"
        T[TENANT<br/>Organization] --> C[CUSTOMER<br/>Sub-org]
        T --> U[USER<br/>Operator]
        C --> U
        T --> D[DEVICE<br/>IoT device]
        T --> A[ASSET<br/>Logical group]
        D --> ALM[ALARM<br/>Alert]
        A --> ALM
    end

    style T fill:#e3f2fd
    style D fill:#e8f5e9
    style ALM fill:#ffcdd2
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| TENANT | 1 | tenant | Multi-tenant organization |
| CUSTOMER | 2 | customer | Sub-organization within tenant |
| USER | 3 | tb_user | Platform user account |
| DEVICE | 6 | device | IoT device or sensor |
| ASSET | 5 | asset | Logical grouping or physical asset |
| DASHBOARD | 4 | dashboard | Visualization dashboard |
| ALARM | 7 | alarm | Alert or notification trigger |

### Rule Engine Entities

Entities that define message processing logic.

```mermaid
graph LR
    subgraph "Rule Engine"
        RC[RULE_CHAIN] -->|contains| RN1[RULE_NODE]
        RC -->|contains| RN2[RULE_NODE]
        RC -->|contains| RN3[RULE_NODE]
        RN1 -->|routes to| RN2
        RN2 -->|routes to| RN3
    end

    style RC fill:#f3e5f5
    style RN1 fill:#e1bee7
    style RN2 fill:#e1bee7
    style RN3 fill:#e1bee7
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| RULE_CHAIN | 11 | rule_chain | Message processing pipeline |
| RULE_NODE | 12 | rule_node | Individual processing step |

### Profile Entities

Templates that define behavior for other entities.

```mermaid
graph TB
    subgraph "Profiles as Templates"
        TP[TENANT_PROFILE<br/>Quotas & limits]
        DP[DEVICE_PROFILE<br/>Transport, alarms]
        AP[ASSET_PROFILE<br/>Rule chain]
    end

    subgraph "Entities Using Profiles"
        T1[Tenant 1] -.->|uses| TP
        T2[Tenant 2] -.->|uses| TP
        D1[Device 1] -.->|uses| DP
        D2[Device 2] -.->|uses| DP
        A1[Asset 1] -.->|uses| AP
    end

    style TP fill:#fff3e0
    style DP fill:#fff3e0
    style AP fill:#fff3e0
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| TENANT_PROFILE | 20 | tenant_profile | Tenant limits and configuration |
| DEVICE_PROFILE | 21 | device_profile | Device behavior template |
| ASSET_PROFILE | 22 | asset_profile | Asset behavior template |

### UI Entities

Entities for visualization and presentation.

```mermaid
graph TB
    subgraph "Dashboard System"
        DASH[DASHBOARD<br/>Container]
        WB[WIDGETS_BUNDLE<br/>Widget collection]
        WT[WIDGET_TYPE<br/>Widget definition]
        EV[ENTITY_VIEW<br/>Filtered data]
    end

    DASH -->|uses widgets from| WB
    WB -->|contains| WT
    DASH -->|displays| EV

    style DASH fill:#fce4ec
    style WB fill:#f8bbd9
    style WT fill:#f48fb1
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| ENTITY_VIEW | 15 | entity_view | Filtered view of entity data |
| WIDGETS_BUNDLE | 16 | widgets_bundle | Collection of widget types |
| WIDGET_TYPE | 17 | widget_type | Reusable dashboard widget |

### System Entities

Platform configuration and management.

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| API_USAGE_STATE | 23 | api_usage_state | Usage tracking per tenant |
| TB_RESOURCE | 24 | resource | Uploaded files and resources |
| OTA_PACKAGE | 25 | ota_package | Firmware/software packages |
| QUEUE | 28 | queue | Message queue configuration |
| QUEUE_STATS | 34 | queue_stats | Queue statistics |
| ADMIN_SETTINGS | 42 | admin_settings | System configuration |

### Edge Computing

```mermaid
graph LR
    subgraph "Cloud"
        TB[ThingsBoard Server]
    end

    subgraph "Edge Sites"
        E1[EDGE: Site A]
        E2[EDGE: Site B]
    end

    subgraph "RPC Communication"
        RPC1[RPC Request]
        RPC2[RPC Response]
    end

    TB <-->|sync| E1
    TB <-->|sync| E2
    E1 -->|execute| RPC1
    RPC1 -->|result| RPC2

    style TB fill:#e3f2fd
    style E1 fill:#c8e6c9
    style E2 fill:#c8e6c9
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| EDGE | 26 | edge | Edge gateway instance |
| RPC | 27 | rpc | Remote procedure call record |

### Notification System

```mermaid
graph LR
    subgraph "Notification Flow"
        NR[NOTIFICATION_RULE<br/>Trigger condition]
        NT[NOTIFICATION_TEMPLATE<br/>Message format]
        NTG[NOTIFICATION_TARGET<br/>Recipients]
        NRQ[NOTIFICATION_REQUEST<br/>Pending]
        N[NOTIFICATION<br/>Sent]
    end

    NR -->|triggers| NRQ
    NT -->|formats| NRQ
    NTG -->|receives| NRQ
    NRQ -->|becomes| N

    style NR fill:#bbdefb
    style N fill:#c8e6c9
```

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| NOTIFICATION_TARGET | 29 | notification_target | Notification recipients |
| NOTIFICATION_TEMPLATE | 30 | notification_template | Message template |
| NOTIFICATION_REQUEST | 31 | notification_request | Pending notification |
| NOTIFICATION | 32 | notification | Sent notification |
| NOTIFICATION_RULE | 33 | notification_rule | Automation rules |

### OAuth & Security

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| OAUTH2_CLIENT | 35 | oauth2_client | OAuth2 provider configuration |
| DOMAIN | 36 | domain | Custom domain mapping |
| API_KEY | 44 | api_key | API access credentials |

### Mobile Applications

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| MOBILE_APP | 37 | mobile_app | Mobile application config |
| MOBILE_APP_BUNDLE | 38 | mobile_app_bundle | Mobile app package |

### Advanced Features (v4.x)

| Entity Type | Proto # | Table Name | Description |
|-------------|---------|------------|-------------|
| CALCULATED_FIELD | 39 | calculated_field | Computed/derived values |
| JOB | 41 | job | Background job execution |
| AI_MODEL | 43 | ai_model | AI/ML model configuration |

---

## Entity Hierarchy

Understanding how entities are organized is crucial for working with ThingsBoard.

```mermaid
graph TB
    subgraph "System Level (Platform-wide)"
        SYS[System Tenant<br/>NULL_UUID]
        SA[System Admin<br/>Full access]
        TP[Tenant Profiles<br/>Quotas]
    end

    subgraph "Tenant Level (Organization)"
        T[Tenant<br/>Isolated organization]
        TA[Tenant Admin<br/>Tenant-wide access]
        DP[Device Profiles]
        AP[Asset Profiles]
        RC[Rule Chains]
    end

    subgraph "Customer Level (Sub-organization)"
        C[Customer<br/>Subdivision]
        CU[Customer User<br/>Limited access]
    end

    subgraph "Entity Level (Data objects)"
        D[Devices]
        A[Assets]
        DASH[Dashboards]
        EV[Entity Views]
    end

    SYS --> T
    SA -.->|creates| TP
    TP -.->|configures| T
    T --> TA
    T --> C
    T --> DP
    T --> AP
    T --> RC
    C --> CU
    T --> D
    T --> A
    T --> DASH
    T --> EV
    D -.->|uses| DP
    A -.->|uses| AP

    style SYS fill:#e1f5fe
    style T fill:#e3f2fd
    style C fill:#e8f5e9
    style D fill:#fff3e0
```

### Access Control by Level

```mermaid
graph TB
    subgraph "Who Can See What"
        SA[System Admin] -->|all tenants| T1[Tenant A]
        SA -->|all tenants| T2[Tenant B]

        TA[Tenant Admin A] -->|own tenant only| T1
        TA -->|blocked| T2

        CU[Customer User] -->|assigned only| D1[Device 1]
        CU -->|blocked| D2[Device 2]
    end

    style SA fill:#ffcdd2
    style TA fill:#fff9c4
    style CU fill:#c8e6c9
```

---

## Entity Identification

### How Entity IDs Work

Every entity in ThingsBoard has a unique identifier composed of two parts:

```mermaid
graph LR
    subgraph "Entity ID Structure"
        ID[Entity ID]
        ID --> TYPE[Entity Type<br/>e.g., DEVICE]
        ID --> UUID[UUID<br/>784f3940-2f04-...]
    end

    subgraph "Serialized Form"
        JSON["{<br/>  entityType: DEVICE,<br/>  id: 784f3940-...<br/>}"]
    end

    ID --> JSON

    style ID fill:#e3f2fd
    style TYPE fill:#fff3e0
    style UUID fill:#e8f5e9
```

### ID Structure Examples

```json
// Device ID
{
  "entityType": "DEVICE",
  "id": "784f3940-2f04-11ec-8f2e-4d7a8c12df56"
}

// Tenant ID
{
  "entityType": "TENANT",
  "id": "a1b2c3d4-5678-90ab-cdef-1234567890ab"
}

// Alarm ID
{
  "entityType": "ALARM",
  "id": "deadbeef-cafe-babe-face-123456789012"
}
```

### Special IDs

```mermaid
graph TB
    subgraph "Special Entity IDs"
        NULL[NULL_UUID<br/>13814000-1dd2-11b2-8080-808080808080<br/>Used for system-level entities]
        SYS_TENANT[System Tenant<br/>Same as NULL_UUID<br/>Parent of all tenants]
    end

    style NULL fill:#ffcdd2
    style SYS_TENANT fill:#ffcdd2
```

### Type-Safe ID Classes

Each entity type has its own ID class for compile-time safety:

```mermaid
classDiagram
    class EntityId {
        <<interface>>
        +getId() UUID
        +getEntityType() EntityType
    }

    class TenantId {
        +TenantId(UUID id)
    }

    class DeviceId {
        +DeviceId(UUID id)
    }

    class AssetId {
        +AssetId(UUID id)
    }

    class AlarmId {
        +AlarmId(UUID id)
    }

    EntityId <|-- TenantId
    EntityId <|-- DeviceId
    EntityId <|-- AssetId
    EntityId <|-- AlarmId
```

---

## Common Entity Interfaces

Entities share common behaviors through interfaces.

### Interface Hierarchy

```mermaid
graph TB
    subgraph "Core Interfaces"
        HTI[HasTenantId<br/>Scoped to tenant]
        HN[HasName<br/>Human-readable name]
        HV[HasVersion<br/>Optimistic locking]
        EE[ExportableEntity<br/>Import/export support]
        HCI[HasCustomerId<br/>Customer assignment]
    end

    subgraph "Entities Implementing"
        D[Device] --> HTI
        D --> HN
        D --> HV
        D --> EE
        D --> HCI

        A[Asset] --> HTI
        A --> HN
        A --> HV
        A --> EE
        A --> HCI
    end

    style HTI fill:#e3f2fd
    style HN fill:#e8f5e9
    style HV fill:#fff3e0
```

### HasTenantId

All tenant-scoped entities implement this interface:

```
Behavior:
- Returns the tenant that owns this entity
- Used for access control checks
- Ensures tenant isolation in queries

Entities: Device, Asset, Customer, Dashboard, RuleChain, and most others
```

### HasName

Entities with human-readable names:

```
Behavior:
- Provides a display name for the entity
- Usually unique within a scope (tenant + type)
- Used in search and filtering

Entities: Device, Asset, Customer, Tenant, RuleChain
```

### HasVersion

Entities with optimistic locking:

```
Behavior:
- Version number increments on each update
- Prevents concurrent modification conflicts
- If versions don't match, update is rejected

Entities: Almost all entities
```

### ExportableEntity

Entities that can be exported/imported:

```
Behavior:
- Supports externalId for cross-system identification
- Enables backup and migration
- Preserves references during import

Entities: Device, Asset, Dashboard, RuleChain
```

---

## Entity Relationships

Entities connect through the relation system, forming a graph structure.

### Relation Basics

```mermaid
graph LR
    subgraph "Relation Structure"
        FROM[From Entity<br/>Source]
        TO[To Entity<br/>Target]
        TYPE[Relation Type<br/>e.g., Contains]
    end

    FROM -->|TYPE| TO

    style FROM fill:#e3f2fd
    style TO fill:#e8f5e9
    style TYPE fill:#fff3e0
```

### Common Relation Patterns

```mermaid
graph TB
    subgraph "Hierarchy: Building > Floor > Room > Sensor"
        B[Asset: Building] -->|Contains| F1[Asset: Floor 1]
        B -->|Contains| F2[Asset: Floor 2]
        F1 -->|Contains| R1[Asset: Room 101]
        F1 -->|Contains| R2[Asset: Room 102]
        R1 -->|Contains| S1[Device: Sensor 1]
        R1 -->|Contains| S2[Device: Sensor 2]
    end

    style B fill:#e3f2fd
    style F1 fill:#bbdefb
    style F2 fill:#bbdefb
    style R1 fill:#90caf9
    style R2 fill:#90caf9
    style S1 fill:#e8f5e9
    style S2 fill:#e8f5e9
```

```mermaid
graph LR
    subgraph "Gateway Pattern"
        GW[Device: Gateway] -->|Manages| D1[Device: Child 1]
        GW -->|Manages| D2[Device: Child 2]
        GW -->|Manages| D3[Device: Child 3]
    end

    style GW fill:#fff9c4
    style D1 fill:#e8f5e9
    style D2 fill:#e8f5e9
    style D3 fill:#e8f5e9
```

```mermaid
graph LR
    subgraph "Customer Assignment"
        C[Customer: Acme] -->|Assigned| D1[Device: Sensor A]
        C -->|Assigned| D2[Device: Sensor B]
        C -->|Assigned| DASH[Dashboard: Overview]
    end

    style C fill:#f3e5f5
    style D1 fill:#e8f5e9
    style D2 fill:#e8f5e9
    style DASH fill:#fce4ec
```

### Relation Types Reference

| Type | Direction | Use Case |
|------|-----------|----------|
| Contains | FROM → TO | Parent contains child (building → floor) |
| Manages | FROM → TO | Gateway manages child device |
| Uses | FROM → TO | Entity uses another entity |
| ManagedByEdge | TO ← FROM | Edge manages entity |

### Relation Type Groups

| Group | Purpose |
|-------|---------|
| COMMON | General-purpose relations |
| ALARM | Alarm propagation relations |
| DASHBOARD | Dashboard-specific relations |
| RULE_CHAIN | Rule chain connections |

---

## Proto Number Mapping

Entity types have stable proto numbers for efficient serialization in gRPC communication between services.

```mermaid
graph LR
    subgraph "Serialization"
        ET[EntityType.DEVICE] -->|to proto| NUM[6]
        NUM -->|to wire| BYTES[0x06]
        BYTES -->|from wire| NUM2[6]
        NUM2 -->|from proto| ET2[EntityType.DEVICE]
    end
```

### Why Proto Numbers?

1. **Compact**: Integer is smaller than string "DEVICE"
2. **Fast**: Integer comparison is faster
3. **Stable**: Numbers don't change between versions
4. **Compatible**: New types get new numbers, old ones preserved

---

## Entity Lifecycle

All entities follow a similar lifecycle pattern:

```mermaid
stateDiagram-v2
    [*] --> Created: Create API call
    Created --> Active: Normal operation
    Active --> Updated: Update API call
    Updated --> Active: Changes saved
    Active --> Deleted: Delete API call
    Deleted --> [*]: Removed from database

    note right of Created
        Entity assigned UUID
        Version = 0
        created_time set
    end note

    note right of Updated
        Version incremented
        Optimistic lock check
    end note

    note right of Deleted
        Cascade delete relations
        Clean up references
    end note
```

---

## Quick Reference

### Entity Types by Count

| Category | Count | Examples |
|----------|-------|----------|
| Core | 7 | Tenant, Device, Asset, Alarm |
| Profiles | 3 | TenantProfile, DeviceProfile, AssetProfile |
| Rule Engine | 2 | RuleChain, RuleNode |
| UI | 3 | Dashboard, WidgetsBundle, WidgetType |
| Notifications | 5 | NotificationRule, Notification, etc. |
| System | 6 | Queue, OtaPackage, Resource, etc. |
| Edge | 2 | Edge, RPC |
| Security | 3 | OAuth2Client, Domain, ApiKey |
| Mobile | 2 | MobileApp, MobileAppBundle |
| Advanced | 3 | CalculatedField, Job, AIModel |

### Most Common Operations by Entity

| Entity | Common Operations |
|--------|-------------------|
| Device | Create, getCredentials, postTelemetry, getRpc |
| Asset | Create, getRelations, getTelemetry |
| Alarm | Create, acknowledge, clear, assign |
| Dashboard | Create, assign to customer, getWidgets |
| RuleChain | Create, setRoot, getNodes |

---

## See Also

- [Device Entity](./device.md) - Device-specific documentation
- [Asset Entity](./asset.md) - Asset-specific documentation
- [Alarm Entity](./alarm.md) - Alarm-specific documentation
- [Entity Relations](./relations.md) - Relation system details
- [Entity IDs](../identity/entity-ids.md) - ID system deep dive
- [Multi-Tenancy](../../01-architecture/multi-tenancy.md) - Tenant isolation
- [Database Schema](../../07-data-persistence/database-schema.md) - Storage details
