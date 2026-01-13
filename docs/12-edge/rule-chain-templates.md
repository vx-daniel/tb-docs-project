# Rule Chain Templates

## Overview

Rule chain templates are server-defined processing pipelines designed for deployment to Edge instances. Unlike standard rule chains that execute on the server, templates serve as blueprints that are provisioned to edges where they run locally. Starting with Edge 4.0, rule chains can also be created directly on the edge instance. Templates enable centralized management of edge processing logic across distributed deployments.

## Key Behaviors

1. **Server-Side Definition**: Templates are created and edited on the ThingsBoard server.

2. **Edge Deployment**: Templates are assigned to specific edge instances for execution.

3. **Local Execution**: Once deployed, rule chains process data locally on the edge.

4. **Synchronized Updates**: Template changes on the server propagate to assigned edges.

5. **Edge-Local Creation**: Edge 4.0+ supports creating rule chains directly on the edge.

## Template vs Standard Rule Chain

```mermaid
graph TB
    subgraph "Standard Rule Chain"
        S_DEF[Define on Server] --> S_RUN[Runs on Server]
        S_RUN --> S_PROC[Process Device Data]
    end

    subgraph "Rule Chain Template"
        T_DEF[Define on Server] --> T_ASSIGN[Assign to Edge]
        T_ASSIGN --> T_SYNC[Sync to Edge]
        T_SYNC --> T_RUN[Runs on Edge]
        T_RUN --> T_PROC[Process Device Data Locally]
    end
```

| Aspect | Standard Rule Chain | Rule Chain Template |
|--------|--------------------|--------------------|
| Defined on | Server | Server |
| Executes on | Server | Edge |
| Location | Rule Chains menu | Edge Management > Rule Chain Templates |
| Scope | All server devices | Assigned edge devices |
| Data location | Cloud | Edge (local) |

## Template Management

### Creating Templates on Server

Templates are managed in the **Edge Management > Rule Chain Templates** section:

```mermaid
sequenceDiagram
    participant Admin
    participant Server as ThingsBoard Server
    participant Edge as ThingsBoard Edge

    Admin->>Server: Create rule chain template
    Admin->>Server: Configure nodes and flow
    Admin->>Server: Save template

    Admin->>Server: Assign template to Edge
    Server->>Edge: Sync template (gRPC)
    Edge->>Edge: Create local rule chain
    Edge-->>Server: Acknowledge receipt
```

### Creating Rule Chains on Edge (v4.0+)

Edge instances can create local rule chains:

```mermaid
sequenceDiagram
    participant Admin
    participant Edge as ThingsBoard Edge
    participant Server as ThingsBoard Server

    Admin->>Edge: Login to Edge UI
    Admin->>Edge: Create new rule chain
    Admin->>Edge: Configure nodes
    Admin->>Edge: Save rule chain

    Note over Edge: Rule chain runs locally
    Note over Server: Not synced to server
```

## Template Assignment

### Assigning to Edge

```mermaid
graph TB
    subgraph "Server"
        TEMPLATE[Rule Chain Template]
        EDGE_LIST[Edge Instances]
    end

    subgraph "Assignment"
        ASSIGN[Assign Template]
    end

    subgraph "Edge A"
        RC_A[Rule Chain Copy]
    end

    subgraph "Edge B"
        RC_B[Rule Chain Copy]
    end

    TEMPLATE --> ASSIGN
    EDGE_LIST --> ASSIGN
    ASSIGN --> RC_A
    ASSIGN --> RC_B
```

### Managing Edge Rule Chains

From the server, navigate to:
1. **Edge Management > Instances**
2. Select edge instance
3. Click **Manage Edge Rule Chains**
4. Assign or unassign templates

## Edge-Specific Rule Nodes

Templates can use edge-specific nodes for cloud communication:

### Push to Cloud Node

Forwards messages or telemetry to the cloud server:

```mermaid
graph LR
    INPUT[Device Data] --> FILTER{Filter}
    FILTER -->|Important| PUSH[Push to Cloud]
    FILTER -->|Local Only| SAVE[Save Locally]
    PUSH --> CLOUD[Cloud Server]
    SAVE --> DB[(Local DB)]
```

**Configuration:**

| Parameter | Description |
|-----------|-------------|
| Scope | TIMESERIES, ATTRIBUTES, or ENTITY |
| Entity types | Filter by entity type |
| Keys | Specific keys to push |

**Use Cases:**
- Forward aggregated data to cloud
- Push alarms to central monitoring
- Sync selected telemetry

### Push to Edge Node (Server-Side)

Used in server rule chains to send data to edge:

```mermaid
graph LR
    SERVER[Server Rule Chain] --> PUSH[Push to Edge]
    PUSH --> EDGE[Edge Instance]
    EDGE --> DEVICE[Device on Edge]
```

**Use Cases:**
- Send configuration updates
- Push server-side calculations to edge
- Distribute commands from cloud

## Common Template Patterns

### Local Processing with Cloud Sync

Process data locally, push summaries to cloud:

```mermaid
graph TB
    INPUT[Input] --> SAVE[Save Telemetry]
    SAVE --> CHECK{Threshold?}
    CHECK -->|Exceeded| ALARM[Create Alarm]
    CHECK -->|Normal| END1[End]
    ALARM --> PUSH[Push to Cloud]
    PUSH --> END2[End]
```

### Data Filtering and Aggregation

Reduce cloud traffic by filtering and aggregating:

```mermaid
graph TB
    INPUT[Input] --> SCRIPT[Transform Script]
    SCRIPT --> WINDOW[Time Window<br/>1 minute]
    WINDOW --> AGG[Calculate Avg/Min/Max]
    AGG --> PUSH[Push to Cloud]
    INPUT --> LOCAL[Save Raw Locally]
```

### Offline-Capable Alarming

Handle alarms locally even when cloud is unavailable:

```mermaid
graph TB
    INPUT[Telemetry] --> PROFILE[Device Profile]
    PROFILE -->|Alarm| LOCAL_ALARM[Local Alarm]
    PROFILE -->|No Alarm| SAVE[Save]

    LOCAL_ALARM --> NOTIFY[Local Notification]
    LOCAL_ALARM --> RPC[RPC to Device]
    LOCAL_ALARM --> PUSH[Push to Cloud]

    NOTIFY --> END1[End]
    RPC --> END2[End]
```

### Gateway Data Routing

Route data from gateway-connected devices:

```mermaid
graph TB
    GW[Gateway Input] --> SWITCH[Device Type Switch]
    SWITCH -->|Sensor| SENSOR_CHAIN[Sensor Processing]
    SWITCH -->|Actuator| ACTUATOR_CHAIN[Actuator Processing]
    SWITCH -->|Unknown| LOG[Log Warning]

    SENSOR_CHAIN --> SAVE1[Save Telemetry]
    ACTUATOR_CHAIN --> SAVE2[Save Attributes]
```

## Template Synchronization

### Initial Sync

When a template is first assigned:

```mermaid
sequenceDiagram
    participant Server
    participant Edge

    Server->>Server: Serialize rule chain
    Server->>Edge: RuleChainUpdateMsg
    Edge->>Edge: Validate template
    Edge->>Edge: Create rule chain
    Edge->>Edge: Start actors
    Edge-->>Server: Acknowledgment
```

### Update Propagation

When a template is modified:

```mermaid
sequenceDiagram
    participant Admin
    participant Server
    participant Edge

    Admin->>Server: Update template
    Server->>Server: Increment version
    Server->>Edge: RuleChainUpdateMsg (delta)

    Edge->>Edge: Stop current chain
    Edge->>Edge: Apply updates
    Edge->>Edge: Restart chain

    Edge-->>Server: Acknowledgment
```

### Conflict Resolution

If edge has local modifications (v4.0+):

| Scenario | Resolution |
|----------|------------|
| Server template updated | Server version overwrites |
| Edge-local chain | Remains unchanged |
| Both modified | Server template takes precedence |

## Root Rule Chain

Each edge has a root rule chain that processes all incoming device messages:

```mermaid
graph TB
    subgraph "Root Rule Chain"
        INPUT[Device Message] --> SWITCH[Message Type Switch]
        SWITCH -->|Telemetry| TEL[Telemetry Processing]
        SWITCH -->|Attributes| ATTR[Attributes Processing]
        SWITCH -->|RPC| RPC[RPC Processing]
        SWITCH -->|Other| DEFAULT[Default Handler]
    end
```

**Root Chain Behavior:**
- Automatically assigned to edge
- First chain to process all device data
- Can route to other rule chains
- Marked with "Root" flag

## Configuration Best Practices

### Design for Offline

- Avoid nodes that require cloud connectivity in critical paths
- Use "Push to Cloud" selectively, not for every message
- Implement local fallback for alarming and notifications

### Optimize for Resources

- Keep rule chains simple on resource-constrained edges
- Avoid complex JavaScript in transform nodes
- Use built-in nodes instead of scripts when possible

### Manage Template Versions

- Test templates on a single edge before broad deployment
- Use naming conventions (e.g., "v1.0 - Temperature Monitor")
- Document template purposes and configurations

## Debugging Templates

### Enable Debug Mode

Debug mode captures message flow on the edge:

```mermaid
graph LR
    MSG[Message] --> NODE[Rule Node]
    NODE --> DEBUG{Debug Mode?}
    DEBUG -->|Yes| CAPTURE[Capture Snapshot]
    DEBUG -->|No| SKIP[Skip]
    CAPTURE --> STORE[(Debug Events)]
    NODE --> NEXT[Next Node]
```

**Captured Information:**
- Input message
- Output message
- Relation type
- Processing time
- Errors

### Debug on Edge vs Cloud

| Debug Location | Visibility | Storage |
|----------------|------------|---------|
| Edge UI | Edge admin only | Local database |
| Server UI (template) | Server admin | Template definition only |

## See Also

- [Edge Architecture](./edge-architecture.md) - Component overview
- [Cloud Synchronization](./cloud-synchronization.md) - Sync protocol
- [Rule Engine](../04-rule-engine/README.md) - Rule chain concepts
- [Node Categories](../04-rule-engine/node-categories.md) - Available nodes
