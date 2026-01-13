# Database Schema

## Overview

ThingsBoard uses a relational database (PostgreSQL) as its primary data store for entities, relationships, credentials, and configuration. Time-series data can be stored in PostgreSQL (using native tables or TimescaleDB extension) or optionally in Apache Cassandra for high-throughput scenarios. The schema follows a multi-tenant architecture where most entities are scoped to a tenant and use UUIDs as primary keys.

## Key Behaviors

1. **UUID Primary Keys**: All entities use UUID as the primary identifier, supporting distributed ID generation.

2. **Tenant Scoping**: Most tables include a `tenant_id` column for multi-tenant isolation.

3. **Optimistic Locking**: Version columns enable concurrent modification detection.

4. **JSON Storage**: Complex configurations are stored as JSONB columns in PostgreSQL.

5. **Dual Time-Series Storage**: Latest values and historical data are stored in separate tables for query optimization.

6. **Key Dictionary**: Time-series keys are mapped to integer IDs for storage efficiency.

## Schema Architecture

### Database Types

```mermaid
graph TB
    subgraph "Entity Storage"
        PG[(PostgreSQL)]
    end

    subgraph "Time-Series Storage"
        TS_SQL[(PostgreSQL)]
        TS_SCALE[(TimescaleDB)]
        TS_CASS[(Cassandra)]
    end

    subgraph "Tables"
        ENTITIES[Entity Tables]
        ATTR[Attribute Tables]
        TS_KV[Time-Series Tables]
    end

    PG --> ENTITIES
    PG --> ATTR
    TS_SQL --> TS_KV
    TS_SCALE --> TS_KV
    TS_CASS --> TS_KV
```

### Core Entity Relationships

```mermaid
erDiagram
    tenant ||--o{ customer : has
    tenant ||--o{ device : owns
    tenant ||--o{ asset : owns
    tenant ||--o{ dashboard : owns
    tenant ||--o{ rule_chain : owns
    tenant }o--|| tenant_profile : uses

    customer ||--o{ device : assigned
    customer ||--o{ asset : assigned

    device }o--|| device_profile : uses
    device ||--|| device_credentials : has

    asset }o--|| asset_profile : uses

    device_profile }o--o| rule_chain : "default chain"
    device_profile }o--o| ota_package : firmware
    device_profile }o--o| ota_package : software

    rule_chain ||--o{ rule_node : contains
```

## Core Tables

### User and Authentication ER Diagram

```mermaid
erDiagram
    tenant ||--o{ tb_user : "has users"
    customer ||--o{ tb_user : "has users"
    tb_user ||--|| user_credentials : "has credentials"
    tb_user ||--o{ user_auth_settings : "has 2FA"

    tb_user {
        uuid id PK
        uuid tenant_id FK
        uuid customer_id FK
        varchar authority
        varchar email UK
        varchar first_name
        varchar last_name
    }

    user_credentials {
        uuid id PK
        uuid user_id FK,UK
        boolean enabled
        varchar password
        varchar activate_token
        bigint last_login_ts
        int failed_login_attempts
    }

    user_auth_settings {
        uuid id PK
        uuid user_id FK
        varchar two_fa_auth_type
    }
```

### tenant

Tenant is the top-level organizational unit.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Tenant UUID |
| created_time | bigint | No | Creation timestamp (ms) |
| title | varchar(255) | No | Tenant name |
| region | varchar(255) | Yes | Geographic region |
| tenant_profile_id | uuid | FK | Profile reference |
| country | varchar(255) | Yes | Contact country |
| state | varchar(255) | Yes | Contact state |
| city | varchar(255) | Yes | Contact city |
| address | varchar(255) | Yes | Contact address |
| address2 | varchar(255) | Yes | Additional address |
| zip | varchar(255) | Yes | ZIP code |
| phone | varchar(255) | Yes | Contact phone |
| email | varchar(255) | Yes | Contact email |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

### tenant_profile

Defines tenant quotas and features.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Profile UUID |
| created_time | bigint | No | Creation timestamp |
| name | varchar(255) | No | Profile name |
| description | varchar | Yes | Description |
| is_default | boolean | No | Default profile flag |
| isolated_tb_rule_engine | boolean | No | Isolated rule engine |
| profile_data | jsonb | No | Configuration (limits, features) |
| version | bigint | No | Optimistic lock version |

### customer

Customer within a tenant, can have assigned devices/assets.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Customer UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Parent tenant |
| title | varchar(255) | No | Customer name |
| is_public | boolean | No | Public customer flag |
| country | varchar(255) | Yes | Contact country |
| state | varchar(255) | Yes | Contact state |
| city | varchar(255) | Yes | Contact city |
| address | varchar(255) | Yes | Contact address |
| address2 | varchar(255) | Yes | Additional address |
| zip | varchar(255) | Yes | ZIP code |
| phone | varchar(255) | Yes | Contact phone |
| email | varchar(255) | Yes | Contact email |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

### tb_user

Platform users (operators, not devices).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | User UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| customer_id | uuid | FK | Associated customer |
| authority | varchar(255) | No | Role (SYS_ADMIN, TENANT_ADMIN, CUSTOMER_USER) |
| email | varchar(255) | No | Login email (unique) |
| first_name | varchar(255) | Yes | First name |
| last_name | varchar(255) | Yes | Last name |
| additional_info | varchar | Yes | JSON metadata |

### user_credentials

User authentication data.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Credentials UUID |
| user_id | uuid | FK | User reference |
| enabled | boolean | No | Account enabled |
| password | varchar(255) | Yes | BCrypt hash |
| activate_token | varchar(255) | Yes | Activation token |
| activate_token_exp_time | bigint | Yes | Token expiration |
| reset_token | varchar(255) | Yes | Password reset token |
| reset_token_exp_time | bigint | Yes | Token expiration |
| last_login_ts | bigint | Yes | Last login timestamp |
| failed_login_attempts | int | Yes | Failed attempts count |

## Device Tables

### Device Subsystem ER Diagram

```mermaid
erDiagram
    tenant ||--o{ device : "owns"
    tenant ||--o{ device_profile : "owns"
    customer ||--o{ device : "assigned"
    device_profile ||--o{ device : "template for"
    device ||--|| device_credentials : "authenticated by"
    device_profile }o--o| ota_package : "firmware"
    device_profile }o--o| ota_package : "software"
    device }o--o| ota_package : "assigned firmware"
    device }o--o| ota_package : "assigned software"

    device {
        uuid id PK
        uuid tenant_id FK
        uuid customer_id FK
        uuid device_profile_id FK
        varchar name UK
        varchar type
        varchar label
        jsonb device_data
    }

    device_profile {
        uuid id PK
        uuid tenant_id FK
        varchar name
        varchar transport_type
        varchar provision_type
        jsonb profile_data
        uuid default_rule_chain_id FK
    }

    device_credentials {
        uuid id PK
        uuid device_id FK,UK
        varchar credentials_type
        varchar credentials_id UK
        varchar credentials_value
    }

    ota_package {
        uuid id PK
        uuid tenant_id FK
        varchar title
        varchar version
        varchar type
        bytea data
    }
```

### device

IoT devices connected to the platform.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Device UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| customer_id | uuid | FK | Assigned customer |
| device_profile_id | uuid | FK | Device profile |
| name | varchar(255) | No | Unique name within tenant |
| type | varchar(255) | Yes | Device type (profile name) |
| label | varchar(255) | Yes | Display label |
| device_data | jsonb | Yes | Transport configuration |
| firmware_id | uuid | FK | Assigned firmware |
| software_id | uuid | FK | Assigned software |
| external_id | uuid | Yes | Import/export ID |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

**Indexes:**
- `device_tenant_id` on (tenant_id)
- `device_customer_id` on (customer_id)
- `device_name_unq_key` UNIQUE on (tenant_id, name)

### device_profile

Device configuration templates.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Profile UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| name | varchar(255) | No | Profile name |
| type | varchar(255) | No | Device type |
| image | varchar | Yes | Profile image |
| transport_type | varchar(255) | No | MQTT, COAP, HTTP, LWM2M, SNMP |
| provision_type | varchar(255) | No | Provisioning method |
| profile_data | jsonb | No | Alarm rules, transport config |
| description | varchar | Yes | Description |
| is_default | boolean | No | Default profile flag |
| default_rule_chain_id | uuid | FK | Default rule chain |
| default_dashboard_id | uuid | FK | Default dashboard |
| default_queue_name | varchar(255) | Yes | Processing queue |
| provision_device_key | varchar(255) | Yes | Provisioning key |
| firmware_id | uuid | FK | Default firmware |
| software_id | uuid | FK | Default software |
| default_edge_rule_chain_id | uuid | FK | Edge rule chain |
| external_id | uuid | Yes | Import/export ID |
| version | bigint | No | Optimistic lock version |

### device_credentials

Device authentication credentials.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Credentials UUID |
| device_id | uuid | FK | Device reference (unique) |
| credentials_type | varchar(255) | No | ACCESS_TOKEN, X509_CERTIFICATE, MQTT_BASIC, LWM2M_CREDENTIALS |
| credentials_id | varchar(255) | No | Token or certificate hash |
| credentials_value | varchar | Yes | Certificate or JSON config |

**Indexes:**
- `device_credentials_id_unq_key` UNIQUE on (credentials_id)
- `device_credentials_device_id_unq_key` UNIQUE on (device_id)

## Asset Tables

### asset

Logical entities for grouping and modeling.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Asset UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| customer_id | uuid | FK | Assigned customer |
| asset_profile_id | uuid | FK | Asset profile |
| name | varchar(255) | No | Unique name within tenant |
| type | varchar(255) | Yes | Asset type (profile name) |
| label | varchar(255) | Yes | Display label |
| external_id | uuid | Yes | Import/export ID |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

### asset_profile

Asset configuration templates.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Profile UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| name | varchar(255) | No | Profile name |
| image | varchar | Yes | Profile image |
| description | varchar | Yes | Description |
| is_default | boolean | No | Default profile flag |
| default_rule_chain_id | uuid | FK | Default rule chain |
| default_dashboard_id | uuid | FK | Default dashboard |
| default_queue_name | varchar(255) | Yes | Processing queue |
| default_edge_rule_chain_id | uuid | FK | Edge rule chain |
| external_id | uuid | Yes | Import/export ID |
| version | bigint | No | Optimistic lock version |

## Relation Table

### relation

Entity-to-entity relationships.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| from_id | uuid | PK | Source entity UUID |
| from_type | varchar(255) | PK | Source entity type |
| to_id | uuid | PK | Target entity UUID |
| to_type | varchar(255) | PK | Target entity type |
| relation_type_group | varchar(255) | PK | Group (COMMON, ALARM, etc.) |
| relation_type | varchar(255) | PK | Relationship name |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

**Primary Key:** (from_id, from_type, to_id, to_type, relation_type_group, relation_type)

## Alarm Tables

### Alarm Subsystem ER Diagram

```mermaid
erDiagram
    tenant ||--o{ alarm : "owns"
    device ||--o{ alarm : "originates"
    asset ||--o{ alarm : "originates"
    tb_user ||--o{ alarm : "assigned to"
    alarm ||--o{ entity_alarm : "propagated to"
    alarm ||--o{ alarm_comment : "has comments"

    alarm {
        uuid id PK
        uuid tenant_id FK
        uuid originator_id
        varchar originator_type
        varchar type
        varchar severity
        boolean acknowledged
        boolean cleared
        bigint start_ts
        bigint end_ts
    }

    entity_alarm {
        uuid tenant_id PK
        varchar entity_type PK
        uuid entity_id PK
        varchar alarm_type PK
        uuid alarm_id FK
    }

    alarm_comment {
        uuid id PK
        uuid alarm_id FK
        uuid user_id FK
        varchar type
        varchar comment
        bigint created_time
    }
```

### Alarm Lifecycle Flow

```mermaid
stateDiagram-v2
    [*] --> Active: Condition triggers
    Active --> Acknowledged: User acknowledges
    Active --> Cleared: Condition clears
    Acknowledged --> AckCleared: Condition clears
    Cleared --> AckCleared: User acknowledges

    state Active {
        [*] --> ACTIVE_UNACK
    }

    state Cleared {
        [*] --> CLEARED_UNACK
    }

    state Acknowledged {
        [*] --> ACTIVE_ACK
    }

    state AckCleared {
        [*] --> CLEARED_ACK
    }

    AckCleared --> [*]: Deleted/Archived
```

### alarm

Active and historical alarms.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Alarm UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| customer_id | uuid | FK | Associated customer |
| originator_id | uuid | No | Source entity UUID |
| originator_type | varchar(255) | No | Source entity type |
| type | varchar(255) | No | Alarm type name |
| severity | varchar(255) | No | CRITICAL, MAJOR, MINOR, WARNING, INDETERMINATE |
| assignee_id | uuid | FK | Assigned user |
| start_ts | bigint | No | Alarm start timestamp |
| end_ts | bigint | Yes | Alarm end timestamp |
| acknowledged | boolean | No | Acknowledged flag |
| ack_ts | bigint | Yes | Acknowledgment timestamp |
| cleared | boolean | No | Cleared flag |
| clear_ts | bigint | Yes | Clear timestamp |
| assign_ts | bigint | Yes | Assignment timestamp |
| additional_info | varchar | Yes | Alarm details JSON |
| propagate | boolean | No | Propagate to related |
| propagate_to_owner | boolean | No | Propagate to owner |
| propagate_to_tenant | boolean | No | Propagate to tenant |
| propagate_relation_types | varchar | Yes | Relation types for propagation |

### entity_alarm

Links alarms to affected entities.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| tenant_id | uuid | PK | Owning tenant |
| entity_type | varchar(255) | PK | Entity type |
| entity_id | uuid | PK | Entity UUID |
| alarm_type | varchar(255) | PK | Alarm type |
| customer_id | uuid | Yes | Customer reference |
| alarm_id | uuid | FK | Alarm reference |
| created_time | bigint | No | Link creation time |

## Rule Engine Tables

### Rule Engine ER Diagram

```mermaid
erDiagram
    tenant ||--o{ rule_chain : "owns"
    rule_chain ||--o{ rule_node : "contains"
    rule_chain ||--|| rule_node : "first node"
    rule_node ||--o{ rule_node_state : "has state"
    device_profile }o--o| rule_chain : "default chain"
    asset_profile }o--o| rule_chain : "default chain"

    rule_chain {
        uuid id PK
        uuid tenant_id FK
        varchar name
        varchar type
        uuid first_rule_node_id FK
        boolean root
        boolean debug_mode
    }

    rule_node {
        uuid id PK
        uuid rule_chain_id FK
        varchar type
        varchar name
        jsonb configuration
        boolean singleton_mode
        varchar queue_name
    }

    rule_node_state {
        uuid id PK
        uuid rule_node_id FK
        uuid entity_id
        varchar state_data
    }
```

### Rule Chain Processing Flow

```mermaid
graph LR
    subgraph "Input"
        MSG[TbMsg] --> FIRST[First Node]
    end

    subgraph "Processing"
        FIRST -->|Success| N2[Node 2]
        FIRST -->|Failure| ERR[Error Handler]
        N2 -->|True| N3A[Node 3A]
        N2 -->|False| N3B[Node 3B]
    end

    subgraph "Output"
        N3A --> SAVE[Save Telemetry]
        N3B --> ALARM[Create Alarm]
    end

    style FIRST fill:#e3f2fd
    style SAVE fill:#e8f5e9
    style ALARM fill:#fff3e0
```

### rule_chain

Processing workflow definitions.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Rule chain UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| name | varchar(255) | No | Chain name |
| type | varchar(255) | No | CORE or EDGE |
| first_rule_node_id | uuid | FK | Entry point node |
| root | boolean | No | Tenant root chain |
| debug_mode | boolean | No | Debug enabled |
| configuration | jsonb | Yes | Reserved for future |
| external_id | uuid | Yes | Import/export ID |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

### rule_node

Processing steps within rule chains.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Rule node UUID |
| created_time | bigint | No | Creation timestamp |
| rule_chain_id | uuid | FK | Parent chain |
| type | varchar(255) | No | Node class name |
| name | varchar(255) | No | Display name |
| configuration | jsonb | No | Node settings |
| configuration_version | int | No | Schema version |
| debug_settings | jsonb | Yes | Debug configuration |
| singleton_mode | boolean | No | Run on single partition |
| queue_name | varchar(255) | Yes | Processing queue |
| external_id | uuid | Yes | Import/export ID |
| additional_info | varchar | Yes | Layout position (x, y) |

## Attribute Storage

### attribute_kv

Entity attribute key-value storage.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| entity_type | varchar(255) | PK | Entity type |
| entity_id | uuid | PK | Entity UUID |
| attribute_type | varchar(255) | PK | Scope (CLIENT, SERVER, SHARED) |
| attribute_key | varchar(255) | PK | Attribute name |
| bool_v | boolean | Yes | Boolean value |
| str_v | varchar(10000000) | Yes | String value |
| long_v | bigint | Yes | Integer value |
| dbl_v | double | Yes | Decimal value |
| json_v | varchar | Yes | JSON value |
| last_update_ts | bigint | No | Last update timestamp |
| version | bigint | No | Optimistic lock version |

**Primary Key:** (entity_type, entity_id, attribute_type, attribute_key)

## Time-Series Storage

### Time-Series Architecture

```mermaid
graph TB
    subgraph "Write Path"
        DEV[Device] -->|telemetry| TS[Transport Service]
        TS -->|message| RE[Rule Engine]
        RE -->|save| DAO[Time-Series DAO]
    end

    subgraph "Storage Layer"
        DAO --> DICT[ts_kv_dictionary<br/>key → key_id mapping]
        DAO --> LATEST[ts_kv_latest<br/>latest values only]
        DAO --> HIST[ts_kv<br/>historical data]
        HIST --> P1[partition_2024_01]
        HIST --> P2[partition_2024_02]
        HIST --> P3[partition_...]
    end

    subgraph "Read Path"
        API[REST API] --> CACHE[Cache Layer]
        CACHE -->|miss| LATEST
        CACHE -->|range query| HIST
    end

    style DICT fill:#e3f2fd
    style LATEST fill:#e8f5e9
    style HIST fill:#fff3e0
```

### Time-Series Storage Pattern

```mermaid
sequenceDiagram
    participant D as Device
    participant P as Platform
    participant Dict as ts_kv_dictionary
    participant L as ts_kv_latest
    participant H as ts_kv (partitioned)

    D->>P: {"temperature": 23.5}
    P->>Dict: lookup "temperature"
    Dict-->>P: key_id = 42

    par Update Latest
        P->>L: UPSERT (entity_id, 42, ts, 23.5)
    and Append History
        P->>H: INSERT (entity_id, 42, ts, 23.5)
    end

    Note over H: Data partitioned by timestamp<br/>Old partitions can be dropped
```

### ts_kv_dictionary

Maps string keys to integer IDs.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| key | varchar(255) | PK | Key name |
| key_id | serial | No | Integer key ID |

### ts_kv

Historical time-series data (partitioned by time).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| entity_id | uuid | PK | Entity UUID |
| key | int | PK | Key ID (from dictionary) |
| ts | bigint | PK | Timestamp (ms) |
| bool_v | boolean | Yes | Boolean value |
| str_v | varchar | Yes | String value |
| long_v | bigint | Yes | Integer value |
| dbl_v | double | Yes | Decimal value |
| json_v | varchar | Yes | JSON value |

**Primary Key:** (entity_id, key, ts)

**Partitioning:** Table is partitioned by timestamp (typically monthly).

### ts_kv_latest

Latest time-series values for fast access.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| entity_id | uuid | PK | Entity UUID |
| key | int | PK | Key ID |
| ts | bigint | No | Last update timestamp |
| bool_v | boolean | Yes | Boolean value |
| str_v | varchar | Yes | String value |
| long_v | bigint | Yes | Integer value |
| dbl_v | double | Yes | Decimal value |
| json_v | varchar | Yes | JSON value |
| version | bigint | No | Optimistic lock version |

**Primary Key:** (entity_id, key)

## Dashboard Tables

### dashboard

User interface definitions.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Dashboard UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| title | varchar(255) | No | Dashboard title |
| image | varchar | Yes | Thumbnail image |
| configuration | jsonb | No | Layout, widgets config |
| assigned_customers | varchar | Yes | JSON array of customer IDs |
| mobile_hide | boolean | No | Hide in mobile app |
| mobile_order | int | Yes | Mobile sort order |
| external_id | uuid | Yes | Import/export ID |
| additional_info | varchar | Yes | JSON metadata |
| version | bigint | No | Optimistic lock version |

## Event Tables

### Event System Architecture

```mermaid
graph TB
    subgraph "Event Sources"
        RE[Rule Engine] -->|errors| ERR[error_event]
        DEV[Device Actor] -->|lifecycle| LC[lc_event]
        SYS[System] -->|metrics| STATS[stats_event]
        RN[Rule Nodes] -->|debug| RND[rule_node_debug_event]
        RC[Rule Chains] -->|debug| RCD[rule_chain_debug_event]
        API[API Layer] -->|actions| AUDIT[audit_log]
    end

    subgraph "Event Storage"
        ERR --> PG[(PostgreSQL)]
        LC --> PG
        STATS --> PG
        RND --> PG
        RCD --> PG
        AUDIT --> PG
        AUDIT -->|optional| ES[(Elasticsearch)]
    end

    subgraph "Retention"
        PG -->|TTL cleanup| DEL[Deleted]
    end

    style ERR fill:#ffcdd2
    style LC fill:#e8f5e9
    style STATS fill:#e3f2fd
    style AUDIT fill:#fff3e0
```

### Events by Type

| Table | Purpose |
|-------|---------|
| error_event | Error events from rule engine |
| lc_event | Lifecycle events (connect/disconnect) |
| stats_event | Statistics events |
| rule_node_debug_event | Rule node debug traces |
| rule_chain_debug_event | Rule chain debug traces |
| audit_log | User action audit trail |

### Common Event Columns

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Event UUID |
| created_time | bigint | Event timestamp |
| tenant_id | uuid | Owning tenant |
| entity_id | uuid | Source entity |
| service_id | varchar | Server instance |
| e_type | varchar | Event type |
| e_entity_id | uuid | Related entity |
| e_entity_type | varchar | Related entity type |
| e_data | varchar | Event data JSON |

## Supporting Tables

### admin_settings

System and tenant configuration.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Setting UUID |
| tenant_id | uuid | FK | Tenant (NULL for system) |
| key | varchar(255) | No | Setting key |
| json_value | varchar | No | Setting value JSON |

### component_descriptor

Available rule node types.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Component UUID |
| type | varchar(255) | No | FILTER, ENRICHMENT, etc. |
| scope | varchar(255) | No | TENANT, SYSTEM |
| name | varchar(255) | No | Display name |
| clazz | varchar(255) | No | Implementation class |
| configuration_descriptor | varchar | No | Config schema JSON |
| configuration_version | int | No | Schema version |
| actions | varchar | Yes | Output relations |
| clustering_mode | varchar(255) | No | ENABLED, SINGLETON |
| has_queue_name | boolean | No | Supports queue config |

### queue

Message queue definitions.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | uuid | PK | Queue UUID |
| created_time | bigint | No | Creation timestamp |
| tenant_id | uuid | FK | Owning tenant |
| name | varchar(255) | No | Queue name |
| topic | varchar(255) | No | Kafka topic |
| poll_interval | int | No | Poll interval (ms) |
| partitions | int | No | Partition count |
| consumer_per_partition | boolean | No | Consumer strategy |
| pack_processing_timeout | bigint | No | Processing timeout |
| submit_strategy | jsonb | No | Submit configuration |
| processing_strategy | jsonb | No | Processing configuration |
| additional_info | varchar | Yes | JSON metadata |

## Index Strategy

### Common Indexes

```sql
-- Tenant scoping
CREATE INDEX idx_device_tenant_id ON device(tenant_id);
CREATE INDEX idx_asset_tenant_id ON asset(tenant_id);

-- Customer assignment
CREATE INDEX idx_device_customer_id ON device(customer_id);

-- Name uniqueness
CREATE UNIQUE INDEX idx_device_name ON device(tenant_id, name);
CREATE UNIQUE INDEX idx_asset_name ON asset(tenant_id, name);

-- Relations lookup
CREATE INDEX idx_relation_from ON relation(from_id, from_type);
CREATE INDEX idx_relation_to ON relation(to_id, to_type);

-- Time-series performance
CREATE INDEX idx_ts_kv_entity_key ON ts_kv(entity_id, key, ts DESC);
```

## Database Configuration

### PostgreSQL Settings

| Setting | Recommended | Description |
|---------|-------------|-------------|
| max_connections | 200-500 | Based on service count |
| shared_buffers | 25% RAM | Shared memory buffer |
| work_mem | 64MB | Query working memory |
| maintenance_work_mem | 512MB | Maintenance operations |
| effective_cache_size | 75% RAM | Query planner hint |

### Connection Pooling (HikariCP)

ThingsBoard uses HikariCP for connection pooling:

| Property | Default | Description |
|----------|---------|-------------|
| spring.datasource.url | jdbc:postgresql://localhost:5432/thingsboard | JDBC URL |
| hikari.leakDetectionThreshold | 0 (disabled) | Leak detection timeout (ms) |
| hikari.maximumPoolSize | 10 | Max connections per pool |
| hikari.minimumIdle | 10 | Minimum idle connections |
| hikari.connectionTimeout | 30000 | Connection timeout (ms) |

### Dedicated Events DataSource

Optional secondary datasource for event data isolation:

```yaml
spring.datasource.events:
  enabled: true  # Enable dedicated events datasource
  url: jdbc:postgresql://localhost:5432/thingsboard_events
```

### Custom PostgreSQL Dialect

ThingsBoard uses a custom Hibernate dialect with additional functions:

```
ThingsboardPostgreSQLDialect
  - ilike() function for case-insensitive search
  - order_by.default_null_ordering: last
```

### TimescaleDB Configuration

For time-series heavy workloads:

```sql
-- Create hypertable
SELECT create_hypertable('ts_kv', 'ts',
  chunk_time_interval => interval '1 day',
  if_not_exists => true);

-- Enable compression
ALTER TABLE ts_kv SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'entity_id,key'
);
```

## Caching Layer

### Caching Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        APP[TB Node]
    end

    subgraph "Local Cache"
        CAF[Caffeine Cache<br/>In-memory, per-node]
    end

    subgraph "Distributed Cache"
        REDIS[(Redis/Valkey<br/>Shared across nodes)]
    end

    subgraph "Database"
        PG[(PostgreSQL)]
    end

    APP -->|1. Check local| CAF
    CAF -->|miss| REDIS
    REDIS -->|miss| PG
    PG -->|populate| REDIS
    REDIS -->|populate| CAF
    CAF -->|hit| APP

    style CAF fill:#e8f5e9
    style REDIS fill:#e3f2fd
    style PG fill:#fff3e0
```

### Cache-Aside Pattern

```mermaid
sequenceDiagram
    participant App as Application
    participant L1 as Caffeine (L1)
    participant L2 as Redis (L2)
    participant DB as PostgreSQL

    App->>L1: get(deviceId)
    alt L1 hit
        L1-->>App: device
    else L1 miss
        App->>L2: get(deviceId)
        alt L2 hit
            L2-->>App: device
            App->>L1: put(deviceId, device)
        else L2 miss
            App->>DB: SELECT * FROM device
            DB-->>App: device
            App->>L2: put(deviceId, device)
            App->>L1: put(deviceId, device)
        end
    end
```

### Redis/Valkey Caching (Distributed)

| Configuration | Default | Description |
|---------------|---------|-------------|
| cache.type | redis | Cache implementation |
| redis.connection.type | standalone | standalone, cluster, sentinel |
| redis.evictTtlInMs | 60000 | Default eviction TTL |
| redis.pool_config.maxTotal | 128 | Max pool connections |
| redis.pool_config.maxIdle | 128 | Max idle connections |
| redis.pool_config.minIdle | 16 | Min idle connections |

### Caffeine Caching (Local)

| Configuration | Default | Description |
|---------------|---------|-------------|
| cache.type | caffeine | In-memory cache |
| Weight-based eviction | Yes | Collision-safe weigher |
| TTL per cache | Configurable | Per-cache specification |
| Stats recording | Enabled | Hit rate tracking |

### Specialized Caches

| Cache | Purpose |
|-------|---------|
| DeviceRedisCache | Device entities |
| CustomerRedisCache | Customer entities |
| UserRedisCache | User entities |
| EdgeRedisCache | Edge entities |
| OtaPackageDataCache | OTA package data |
| TsLatestCache | Latest time-series values |
| UsersSessionInvalidationRedisCache | Token invalidation |

### Time-Series Latest Caching

Redis-backed latest time-series values with cache-aside pattern:

```
CachedRedisSqlTimeseriesLatestDao
  - VersionedTbCache<TsLatestCacheKey, TsKvEntry>
  - Stats tracking: hit/miss counters
```

## Best Practices

1. **UUID Generation**: Use time-based UUIDs (Type 1) for better index locality.

2. **Tenant Isolation**: Always include tenant_id in queries to leverage indexes.

3. **Partition Maintenance**: Regularly clean old time-series partitions.

4. **Connection Pooling**: Configure HikariCP pool sizes based on workload.

5. **JSON Queries**: Create functional indexes for frequently queried JSON paths.

6. **Vacuum Strategy**: Configure autovacuum for tables with high update frequency.

7. **Cache Strategy**: Use Redis for distributed caching, Caffeine for local.

## See Also

- [Time-Series Storage](./timeseries-storage.md) - Detailed time-series patterns
- [Cassandra Storage](./cassandra-storage.md) - Cassandra-specific configuration
- [Caching](./caching.md) - Redis and Caffeine cache configuration
- [Attribute Storage](./attribute-storage.md) - Attribute persistence details
- [Multi-Tenancy](../01-architecture/multi-tenancy.md) - Tenant isolation
- [Telemetry Data Model](../02-core-concepts/data-model/telemetry.md) - Telemetry structure
- [Attributes Data Model](../02-core-concepts/data-model/attributes.md) - Attribute structure
- [Device Entity](../02-core-concepts/entities/device.md) - Device details
