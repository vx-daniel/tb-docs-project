# Cloud Synchronization

## Overview

ThingsBoard Edge synchronizes with the cloud server using a bidirectional gRPC protocol. The Cloud Manager component maintains this connection, handling both outbound events (edge to cloud) and inbound events (cloud to edge). Events are queued locally during connectivity loss and synchronized when the connection is restored, ensuring no data loss.

## Key Behaviors

1. **Bidirectional Sync**: Events flow both from edge to cloud and cloud to edge.

2. **Persistent Queue**: Cloud events are stored locally until successfully delivered.

3. **Automatic Reconnection**: Cloud Manager automatically reconnects after network failures.

4. **Event Ordering**: Events are processed in order within each entity context.

5. **Acknowledgment-Based**: Events remain pending until cloud acknowledges receipt.

## Synchronization Architecture

```mermaid
graph TB
    subgraph "Edge Instance"
        subgraph "Event Sources"
            RE[Rule Engine]
            API[REST API]
            DEVICE[Device Events]
        end

        subgraph "Cloud Manager"
            QUEUE[(Cloud Event Queue)]
            CM[Cloud Manager Service]
            GRPC_C[gRPC Client]
        end
    end

    subgraph "ThingsBoard Cloud"
        GRPC_S[gRPC Server]
        EDGE_SVC[Edge Service]
        CLOUD_QUEUE[(Edge Event Queue)]
    end

    RE --> QUEUE
    API --> QUEUE
    DEVICE --> QUEUE

    QUEUE --> CM
    CM --> GRPC_C
    GRPC_C <-->|gRPC Stream| GRPC_S
    GRPC_S --> EDGE_SVC

    EDGE_SVC --> CLOUD_QUEUE
    CLOUD_QUEUE --> GRPC_S
```

## gRPC Protocol

### Connection Establishment

```mermaid
sequenceDiagram
    participant Edge as Edge Instance
    participant Cloud as ThingsBoard Cloud

    Edge->>Cloud: Connect (routing_key, secret)
    Cloud->>Cloud: Validate credentials
    Cloud-->>Edge: Connection accepted

    par Uplink Stream
        Edge->>Cloud: UplinkMsg (events)
        Cloud-->>Edge: UplinkResponseMsg (ack)
    and Downlink Stream
        Cloud->>Edge: DownlinkMsg (events)
        Edge-->>Cloud: DownlinkResponseMsg (ack)
    end
```

### Message Types

**Uplink Messages (Edge → Cloud):**

| Message Type | Description |
|--------------|-------------|
| EntityDataProto | Entity create/update/delete |
| TelemetryUpdateProto | Time-series data |
| AttributeUpdateProto | Attribute changes |
| AlarmUpdateProto | Alarm state changes |
| RelationUpdateProto | Relation modifications |
| RpcResponseProto | RPC command responses |

**Downlink Messages (Cloud → Edge):**

| Message Type | Description |
|--------------|-------------|
| EntityDataProto | Entity provisioning |
| AttributeUpdateProto | Server/shared attributes |
| RpcRequestProto | RPC commands to devices |
| RuleChainUpdateProto | Rule chain template updates |
| WidgetsBundleUpdateProto | Widget bundle sync |
| DashboardUpdateProto | Dashboard provisioning |

## Cloud Event Types

### Outbound Events (Edge → Cloud)

Events generated on the edge that sync to cloud:

```mermaid
graph TB
    subgraph "Entity Events"
        E_ADD[ADDED]
        E_UPD[UPDATED]
        E_DEL[DELETED]
    end

    subgraph "Data Events"
        D_TS[TIMESERIES_UPDATED]
        D_ATTR[ATTRIBUTES_UPDATED]
        D_ATTR_DEL[ATTRIBUTES_DELETED]
        D_TS_DEL[TIMESERIES_DELETED]
    end

    subgraph "Alarm Events"
        A_ACK[ALARM_ACK]
        A_CLR[ALARM_CLEAR]
    end

    subgraph "Relation Events"
        R_ADD[RELATION_ADD_OR_UPDATE]
        R_DEL[RELATION_DELETED]
        R_DEL_ALL[RELATIONS_DELETED]
    end

    subgraph "Other Events"
        RPC[RPC_CALL]
        CRED[CREDENTIALS_UPDATED]
    end
```

### Event Status

| Status | Description |
|--------|-------------|
| PENDING | Event created, awaiting sync |
| DEPLOYED | Successfully pushed to cloud |

### Cloud Event Table Schema

```sql
CREATE TABLE cloud_event (
    id              UUID PRIMARY KEY,
    created_time    BIGINT NOT NULL,
    tenant_id       UUID NOT NULL,
    edge_id         UUID,
    cloud_event_type VARCHAR(255),
    entity_type     VARCHAR(255),
    entity_id       UUID,
    cloud_event_action VARCHAR(255),
    entity_body     TEXT,
    ts              BIGINT NOT NULL
);
```

## Inbound Events (Cloud → Edge)

Events pushed from cloud to edge instances:

| Event Type | Trigger | Effect on Edge |
|------------|---------|----------------|
| Device provisioned | Admin assigns device to edge | Device created on edge |
| Asset provisioned | Admin assigns asset to edge | Asset created on edge |
| Dashboard provisioned | Admin assigns dashboard | Dashboard synced to edge |
| Rule chain updated | Template modified | Rule chain updated on edge |
| Shared attribute set | Server updates attribute | Attribute synced to device |
| RPC request | Server initiates RPC | Command forwarded to device |
| User provisioned | User assigned to edge | User can access edge UI |

## Sync Mechanisms

### Push Strategy

Events are pushed to cloud as soon as connectivity is available:

```mermaid
sequenceDiagram
    participant Device
    participant Edge as Edge Rule Engine
    participant Queue as Cloud Event Queue
    participant CM as Cloud Manager
    participant Cloud

    Device->>Edge: Post telemetry
    Edge->>Edge: Process rule chain
    Edge->>Queue: Create cloud event (PENDING)

    CM->>Queue: Poll for pending events
    Queue-->>CM: Return batch

    CM->>Cloud: Send batch (gRPC)
    Cloud-->>CM: Acknowledgment

    CM->>Queue: Mark as DEPLOYED
```

### Offline Handling

When connectivity is lost:

```mermaid
stateDiagram-v2
    [*] --> Connected: Startup
    Connected --> Processing: Events arrive
    Processing --> Connected: Events synced

    Connected --> Disconnected: Network failure
    Disconnected --> Queuing: Events arrive
    Queuing --> Queuing: More events

    Disconnected --> Reconnecting: Network restored
    Reconnecting --> Connected: Auth success
    Reconnecting --> Disconnected: Auth failed

    Connected --> Draining: Connection restored
    Draining --> Processing: Queue empty
    Draining --> Draining: More queued events
```

**Queue Behavior During Offline:**

1. Events continue to be generated locally
2. Cloud events stored with PENDING status
3. No data loss - local persistence ensures durability
4. When reconnected, queued events are processed in order

### Batch Processing

Events are sent in batches for efficiency:

| Parameter | Description | Default |
|-----------|-------------|---------|
| batch.size | Max events per batch | 100 |
| batch.timeout | Max wait time (ms) | 1000 |
| retry.interval | Retry delay (ms) | 3000 |
| max.retries | Max retry attempts | 10 |

## Request-Response Patterns

### Attribute Request

Edge can request current attribute values from cloud:

```mermaid
sequenceDiagram
    participant Device
    participant Edge
    participant Cloud

    Device->>Edge: Request shared attributes
    Edge->>Edge: Check local cache
    Edge->>Cloud: Attributes Request

    Cloud->>Cloud: Lookup attributes
    Cloud-->>Edge: Attributes Response

    Edge->>Edge: Update local cache
    Edge-->>Device: Attributes values
```

### Rule Chain Metadata Request

Edge can request rule chain configuration:

```mermaid
sequenceDiagram
    participant Edge
    participant Cloud

    Edge->>Cloud: Rule Chain Metadata Request
    Cloud->>Cloud: Serialize rule chain
    Cloud-->>Edge: Rule Chain Metadata Response
    Edge->>Edge: Update local rule chain
```

## Configuration

### Cloud Connection Settings

| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| CLOUD_ROUTING_KEY | Edge routing key | - |
| CLOUD_ROUTING_SECRET | Edge secret | - |
| CLOUD_RPC_HOST | Cloud server address | localhost |
| CLOUD_RPC_PORT | gRPC port | 7070 |
| CLOUD_RPC_TIMEOUT | Request timeout (ms) | 5000 |
| CLOUD_RPC_SSL_ENABLED | Enable TLS | false |
| CLOUD_RPC_SSL_CERT | TLS certificate path | - |

### Reconnection Settings

| Parameter | Description | Default |
|-----------|-------------|---------|
| reconnect.interval | Base reconnect delay (ms) | 3000 |
| reconnect.max.interval | Max backoff delay (ms) | 60000 |
| keepalive.time | Keepalive interval (s) | 30 |

## Traffic Optimization

### Selective Sync

Not all data needs to go to cloud. Use rule chains to filter:

```mermaid
graph LR
    INPUT[Telemetry] --> FILTER{Important?}
    FILTER -->|Yes| PUSH[Push to Cloud]
    FILTER -->|No| LOCAL[Save Locally]
    LOCAL --> END1[End]
    PUSH --> END2[End]
```

### Data Aggregation

Reduce traffic by aggregating before sync:

```mermaid
graph LR
    RAW[Raw Data<br/>1000 msgs/min] --> AGG[Aggregate<br/>Every 1 min]
    AGG --> SUMMARY[Summary<br/>1 msg/min]
    SUMMARY --> CLOUD[Push to Cloud]
```

**Example Rule Chain for Aggregation:**

1. Collect telemetry in time window
2. Calculate min/max/avg
3. Push aggregated values to cloud
4. Discard raw values

## Monitoring Sync Status

### Cloud Events Page

The Edge UI provides a Cloud Events page showing:
- Event type and action
- Entity information
- Status (PENDING/DEPLOYED)
- Timestamp

### Health Indicators

| Metric | Healthy | Warning |
|--------|---------|---------|
| Queue depth | < 1000 | > 5000 |
| Sync latency | < 5s | > 30s |
| Connection status | Connected | Reconnecting |

## See Also

- [Edge Architecture](./edge-architecture.md) - Component overview
- [Rule Chain Templates](./rule-chain-templates.md) - Template sync
- [Message Queue](../08-message-queue/README.md) - Internal queuing
- [gRPC/Protobuf](../05-transport-layer/README.md) - Protocol details
