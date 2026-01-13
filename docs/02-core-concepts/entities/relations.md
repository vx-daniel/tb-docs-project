# Entity Relations

## Overview

Entity Relations are directed links between any two entities in the platform. They enable hierarchical structures, dependency tracking, and graph-based queries. Relations power critical features including asset hierarchies, alarm propagation, dashboard entity aliases, and rule chain routing.

## Key Behaviors

1. **Directed Links**: Relations have a source (from) and destination (to) entity, creating a directed graph.

2. **Type Categorization**: Each relation has a type (e.g., "Contains") and type group (e.g., COMMON) allowing domain-specific isolation.

3. **Unique Keys**: A relation is uniquely identified by (from, to, type, typeGroup). The same two entities can have multiple relations with different types.

4. **Graph Traversal**: Relations support recursive queries to traverse hierarchies at configurable depths.

5. **Access Controlled**: Users can only see relations where they have read access to both endpoints.

## Data Structure

### Entity Relation

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| from | EntityId | Source entity | Required |
| to | EntityId | Destination entity | Required |
| type | string | Relation type name | Required, max 255 chars |
| typeGroup | enum | Domain category | Required |
| additionalInfo | object | Custom metadata | Optional JSON |
| version | integer | Optimistic locking | Auto-managed |

### Example Relation JSON

```json
{
  "from": {
    "entityType": "ASSET",
    "id": "784f3940-2f04-11ec-8f2e-4d7a8c12df56"
  },
  "to": {
    "entityType": "DEVICE",
    "id": "93a2b1c0-3d5e-11ec-9f8a-1234567890ab"
  },
  "type": "Contains",
  "typeGroup": "COMMON",
  "additionalInfo": {
    "installedDate": "2024-01-15",
    "location": "Floor 2"
  }
}
```

## Direction Semantics

Relations are directional. The direction you query determines what you find:

```mermaid
graph LR
    A[Asset A] -->|Contains| D[Device B]

    subgraph "FROM Direction"
        Q1["findByFrom(A)"] --> R1["Returns: Device B"]
    end

    subgraph "TO Direction"
        Q2["findByTo(B)"] --> R2["Returns: Asset A"]
    end
```

| Direction | Query Meaning | Example |
|-----------|---------------|---------|
| FROM | What does this entity point to? | Find all devices an asset contains |
| TO | What points to this entity? | Find which asset contains a device |

### Direction Example

```mermaid
flowchart TB
    subgraph Hierarchy["Asset A Contains Device B"]
        A[Asset A] -->|"Contains"| B[Device B]
    end

    subgraph Queries["Query Results"]
        F1["findByFrom(A, Contains)"] --> R1["→ Device B"]
        F2["findByTo(B, Contains)"] --> R2["→ Asset A"]
        F3["findByFrom(B, Contains)"] --> R3["→ (empty)"]
        F4["findByTo(A, Contains)"] --> R4["→ (empty)"]
    end
```

## Relation Types

### Built-in Types

| Type | Constant | Purpose |
|------|----------|---------|
| Contains | `CONTAINS_TYPE` | Hierarchical containment |
| Manages | `MANAGES_TYPE` | Management relationship |
| Uses | `USES_TYPE` | Dependency relationship |
| ManagedByEdge | `EDGE_TYPE` | Edge gateway management |

### Custom Types

Any string can be used as a relation type for domain-specific needs:

```
"IsControlledBy"     - Control system relationships
"ProducesDataFor"    - Data flow relationships
"RequiresMaintenance" - Maintenance tracking
"BacksUp"            - Redundancy relationships
"MonitorsHealth"     - Monitoring relationships
```

## Type Groups

Type groups isolate relations by domain, allowing independent operations on different subsystems:

| Group | Purpose | Example Use |
|-------|---------|-------------|
| COMMON | General hierarchies and dependencies | Asset → Device containment |
| DASHBOARD | Dashboard to entity links | Dashboard entity aliases |
| RULE_CHAIN | Rule chain organization | Rule chain to node relations |
| RULE_NODE | Rule node connections | Node success/failure routing |
| EDGE | Edge device management | Edge gateway assignments |
| EDGE_AUTO_ASSIGN_RULE_CHAIN | Auto-assignment rules | Edge rule chain bindings |

```mermaid
graph TB
    subgraph "Same Entities, Different Groups"
        E1[Asset A]
        E2[Device B]
    end

    E1 -->|"Contains<br/>(COMMON)"| E2
    E1 -.->|"Monitors<br/>(DASHBOARD)"| E2

    subgraph Legend
        L1["Solid: Hierarchy relation"]
        L2["Dashed: Dashboard relation"]
    end
```

Relations with the same from/to/type but different type groups are independent and coexist.

## Common Patterns

### Asset/Device Hierarchies

```mermaid
graph TB
    F[Factory<br/>Asset] -->|Contains| B1[Building A<br/>Asset]
    F -->|Contains| B2[Building B<br/>Asset]

    B1 -->|Contains| FL1[Floor 1<br/>Asset]
    B1 -->|Contains| FL2[Floor 2<br/>Asset]

    FL1 -->|Contains| S1[Temp Sensor<br/>Device]
    FL1 -->|Contains| S2[Motion Sensor<br/>Device]

    FL2 -->|Contains| S3[HVAC Controller<br/>Device]

    B2 -->|Contains| P1[Parking Lot<br/>Asset]
    P1 -->|Contains| C1[Charger 1<br/>Device]
    P1 -->|Contains| C2[Charger 2<br/>Device]
```

Query all devices in Building A:
```
findByQuery(
  root: Building A,
  direction: FROM,
  type: "Contains",
  maxLevel: 3,
  entityTypes: [DEVICE]
)
```

### Alarm Propagation

Relations define how alarms escalate through hierarchies:

```mermaid
sequenceDiagram
    participant D as Device
    participant A1 as Parent Asset
    participant A2 as Grandparent Asset
    participant T as Tenant

    D->>D: Alarm triggered
    Note over D: propagate=true

    D->>A1: Find via TO/Contains
    A1->>A1: Propagated alarm created

    A1->>A2: Find via TO/Contains
    A2->>A2: Propagated alarm created

    Note over D,T: Alarm visible at all levels
```

### Dashboard Entity Aliases

Relations link dashboards to data sources:

```mermaid
graph LR
    subgraph Dashboard
        D[My Dashboard]
    end

    subgraph "DASHBOARD Relations"
        D -->|"Uses<br/>(DASHBOARD)"| A1[Production Asset]
        D -->|"Uses<br/>(DASHBOARD)"| A2[Warehouse Asset]
    end

    subgraph "Entity Alias"
        EA["Alias: 'All Production Devices'<br/>Filter: Contains from Production Asset"]
    end

    A1 -->|"Contains<br/>(COMMON)"| DEV1[Device 1]
    A1 -->|"Contains<br/>(COMMON)"| DEV2[Device 2]
```

### Rule Chain Routing

Relations determine execution paths between rule nodes:

```mermaid
graph LR
    RC[Rule Chain] -->|"RULE_CHAIN"| N1[Filter Node]

    N1 -->|"Success<br/>(RULE_NODE)"| N2[Save Telemetry]
    N1 -->|"Failure<br/>(RULE_NODE)"| N3[Log Error]

    N2 -->|"Success<br/>(RULE_NODE)"| N4[Send Notification]
```

## Query Capabilities

### Simple Queries

| Query | Description |
|-------|-------------|
| `findByFrom(entity, typeGroup)` | All relations FROM entity |
| `findByFromAndType(entity, type, typeGroup)` | Relations FROM entity with specific type |
| `findByTo(entity, typeGroup)` | All relations TO entity |
| `findByToAndType(entity, type, typeGroup)` | Relations TO entity with specific type |

### Complex Queries

Use `EntityRelationsQuery` for multi-level graph traversal:

```json
{
  "parameters": {
    "rootId": "asset-uuid",
    "rootType": "ASSET",
    "direction": "FROM",
    "relationTypeGroup": "COMMON",
    "maxLevel": 3,
    "fetchLastLevelOnly": false
  },
  "filters": [
    {
      "relationType": "Contains",
      "entityTypes": ["DEVICE", "ASSET"],
      "negate": false
    }
  ]
}
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| rootId / rootType | Starting entity for the search |
| direction | FROM (outgoing) or TO (incoming) |
| relationTypeGroup | Which domain's relations to search |
| maxLevel | Search depth (1 = direct relations only) |
| fetchLastLevelOnly | Return only entities at maxLevel depth |

### Filters

Narrow results by entity type:

```json
{
  "relationType": "Contains",
  "entityTypes": ["DEVICE", "ASSET"],
  "negate": false
}
```

- `negate: false` - Include only these types
- `negate: true` - Exclude these types

### Query Flow

```mermaid
flowchart TD
    Q[Query Request] --> P[Parse Parameters]
    P --> R[Start from Root Entity]
    R --> L1[Level 1: Find Direct Relations]
    L1 --> F1[Apply Filters]
    F1 --> C1{maxLevel reached?}
    C1 -->|No| L2[Level 2: Find Next Relations]
    L2 --> F2[Apply Filters]
    F2 --> C2{maxLevel reached?}
    C2 -->|No| LN[Continue...]
    C2 -->|Yes| RES[Return Results]
    C1 -->|Yes| RES

    RES --> FLO{fetchLastLevelOnly?}
    FLO -->|Yes| LAST[Return only final level]
    FLO -->|No| ALL[Return all levels]
```

## REST API Endpoints

### Create/Update

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/relation` | POST | Create or update relation |
| `/api/v2/relation` | POST | Create or update (v2 API) |

Request body:
```json
{
  "from": { "entityType": "ASSET", "id": "..." },
  "to": { "entityType": "DEVICE", "id": "..." },
  "type": "Contains",
  "typeGroup": "COMMON",
  "additionalInfo": { "custom": "data" }
}
```

### Delete

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/relation` | DELETE | Delete specific relation |
| `/api/relations` | DELETE | Delete all COMMON relations for entity |

Query parameters for specific deletion:
```
fromId, fromType, toId, toType, relationType, relationTypeGroup
```

### Check Existence

```
GET /api/relation?fromId=...&fromType=...&toId=...&toType=...&relationType=...
```

Returns the relation if it exists, 404 if not.

### Find Relations

| Endpoint | Parameters | Description |
|----------|------------|-------------|
| `GET /api/relations` | fromId, fromType, relationTypeGroup | All FROM entity |
| `GET /api/relations` | fromId, fromType, relationType, relationTypeGroup | FROM with type |
| `GET /api/relations` | toId, toType, relationTypeGroup | All TO entity |
| `GET /api/relations` | toId, toType, relationType, relationTypeGroup | TO with type |

### Complex Queries

```
POST /api/relations
Body: EntityRelationsQuery

POST /api/relations/info
Body: EntityRelationsQuery
Returns: EntityRelationInfo (includes entity names)
```

## EntityRelationInfo

Enhanced response that includes human-readable entity names:

| Field | Description |
|-------|-------------|
| from | Source entity ID |
| to | Destination entity ID |
| type | Relation type |
| typeGroup | Type group |
| fromName | Name of source entity |
| toName | Name of destination entity |
| additionalInfo | Custom metadata |

## Operations

### Create Relation

```mermaid
sequenceDiagram
    participant C as Client
    participant API as REST API
    participant RS as Relation Service
    participant DB as Database

    C->>API: POST /api/relation
    API->>RS: saveRelation(relation)
    RS->>RS: Validate entities exist
    RS->>RS: Check permissions
    RS->>DB: Upsert relation
    DB-->>RS: Saved
    RS->>RS: Invalidate cache
    RS-->>API: Relation
    API-->>C: 200 OK
```

### Delete Patterns

| Operation | Scope |
|-----------|-------|
| `deleteRelation(from, to, type, typeGroup)` | Single relation |
| `deleteEntityRelations(entity)` | ALL relations (both directions, all groups) |
| `deleteEntityCommonRelations(entity)` | Only COMMON group relations |

```mermaid
flowchart TD
    subgraph "deleteEntityRelations"
        E[Entity X]
        R1[Relation A → X]
        R2[Relation X → B]
        R3[Relation X → C]
        R4[Relation D → X]
    end

    DEL[Delete ALL] --> E
    E -.->|Deleted| R1
    E -.->|Deleted| R2
    E -.->|Deleted| R3
    E -.->|Deleted| R4
```

## Access Control

| User Type | Can View | Can Modify |
|-----------|----------|------------|
| Tenant Admin | Relations within tenant | Relations within tenant |
| Customer User | Relations for assigned entities | Relations for assigned entities |
| System Admin | All relations | All relations |

Both entities in a relation must be accessible to the user:
- READ permission on both entities to view
- WRITE permission on both entities to create/delete

## Implementation Patterns

### Building a Hierarchy

```mermaid
sequenceDiagram
    participant A as Admin
    participant API as API

    A->>API: Create Factory asset
    A->>API: Create Building A asset
    A->>API: Create relation: Factory Contains Building A
    A->>API: Create Floor 1 asset
    A->>API: Create relation: Building A Contains Floor 1
    A->>API: Create Device X
    A->>API: Create relation: Floor 1 Contains Device X

    Note over A,API: Hierarchy complete
```

### Finding All Ancestors

Walk up the hierarchy from any entity:

```mermaid
flowchart TD
    D[Device X] -->|"TO / Contains"| A1[Floor 1]
    A1 -->|"TO / Contains"| A2[Building A]
    A2 -->|"TO / Contains"| A3[Factory]
    A3 -->|"No more parents"| STOP[Root reached]
```

Query: `direction=TO, type=Contains, maxLevel=10`

### Finding All Descendants

Walk down the hierarchy from any entity:

```mermaid
flowchart TD
    F[Factory] -->|"FROM / Contains"| B1[Building A]
    F -->|"FROM / Contains"| B2[Building B]
    B1 -->|"FROM / Contains"| FL1[Floor 1]
    B1 -->|"FROM / Contains"| FL2[Floor 2]
    FL1 -->|"FROM / Contains"| D1[Device 1]
    FL1 -->|"FROM / Contains"| D2[Device 2]
```

Query: `direction=FROM, type=Contains, maxLevel=10`

### Impact Analysis

Find all entities that depend on a given entity:

```mermaid
flowchart LR
    E[Entity X]

    E -->|"TO / Uses"| D1[Dependent 1]
    E -->|"TO / Manages"| D2[Dependent 2]
    E -->|"TO / Contains"| D3[Dependent 3]

    D1 -->|"TO / Uses"| D4[Cascade 1]
    D3 -->|"TO / Contains"| D5[Cascade 2]
```

Use TO direction to find what depends on an entity before modifying or deleting it.

## Performance Considerations

### Query Depth

- Use `maxLevel` appropriately - don't set unnecessarily high
- Recursive queries have a configurable timeout (default 20 seconds)
- `fetchLastLevelOnly=true` reduces memory for large hierarchies

### Caching

- Relations are cached automatically
- Cache invalidation occurs on relation changes
- Large hierarchies benefit from caching on repeated queries

### Type Groups

- Use appropriate type groups to isolate domain-specific relations
- Improves query performance (doesn't search irrelevant relations)
- Enables independent operations (delete dashboard relations without affecting hierarchy)

## Edge Cases

### Circular Relations

The platform allows circular relations but queries have depth limits:
- `maxLevel` prevents infinite loops
- Query timeout provides additional protection
- Design hierarchies to avoid unnecessary cycles

### Orphaned Relations

When an entity is deleted:
- Relations are NOT automatically deleted
- Use `deleteEntityRelations()` before entity deletion
- Or implement cleanup logic in your application

### Concurrent Modifications

- Optimistic locking via `version` field prevents lost updates
- Retry logic may be needed for high-contention scenarios

## Implementation Details

### Relation Type Groups

`RelationTypeGroup` enum defines isolation domains:

| Enum Value | Ordinal | Purpose |
|------------|---------|---------|
| `COMMON` | 0 | General entity hierarchies |
| `DASHBOARD` | 1 | Dashboard to entity bindings |
| `RULE_CHAIN` | 2 | Rule chain organization |
| `RULE_NODE` | 3 | Rule node connections |
| `EDGE` | 4 | Edge device management |
| `EDGE_AUTO_ASSIGN_RULE_CHAIN` | 5 | Edge auto-assignment rules |

### Relation Service Architecture

**BaseRelationService** provides core operations:
- Bidirectional index maintenance (from-index and to-index)
- Recursive query with configurable depth and timeout
- Cache invalidation on relation changes
- Bulk operations for entity deletion cleanup

### Caching Strategy

Relations cached at multiple levels:
- **Single relation cache**: Key = `{fromId, toId, type, typeGroup}`
- **From-direction cache**: Key = `{fromId, typeGroup}` → List of relations
- **To-direction cache**: Key = `{toId, typeGroup}` → List of relations
- Cache invalidation broadcasts via cluster notifications

### Recursive Query Implementation

`findByQuery` uses breadth-first traversal:
```
1. Start from root entity
2. For each level (1 to maxLevel):
   a. Query relations in specified direction
   b. Apply entity type filters
   c. Collect results (or just final level if fetchLastLevelOnly)
   d. Use found entities as roots for next level
3. Return accumulated results
```

Timeout protection: `relationsFetchTimeoutInSec` (default: 20 seconds)

### Database Schema

Relations stored in `relation` table:
- Composite primary key: `(from_id, from_type, to_id, to_type, relation_type_group, relation_type)`
- Indexes on both `from_id` and `to_id` for bidirectional queries
- `additional_info` stored as JSONB for custom metadata

### Bulk Operations

**deleteEntityRelations** removes all relations for an entity:
- Queries both FROM and TO directions
- Deletes in batches to avoid lock contention
- Used during entity deletion to prevent orphaned relations

### Configuration Properties

```yaml
sql:
  relations:
    max_level: 50  # Maximum recursion depth
    fetch_timeout_in_sec: 20  # Query timeout

cache:
  relations:
    timeToLiveInMinutes: 1440
    maxSize: 100000
```

## See Also

- [Asset Entity](./asset.md) - Common relation source
- [Device Entity](./device.md) - Common relation target
- [Alarm Entity](./alarm.md) - Alarm propagation via relations
- [Rule Engine](../../04-rule-engine/README.md) - Rule node relations
- [Multi-Tenancy](../../01-architecture/multi-tenancy.md) - Tenant isolation
