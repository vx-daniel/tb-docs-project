# Rule Engine Overview

## Overview

The Rule Engine is the configurable business logic layer of the platform. It processes messages from devices and other sources through user-defined processing pipelines called rule chains. Each rule chain is a directed graph of rule nodes that filter, transform, enrich, and act on messages. The rule engine enables complex IoT workflows without writing custom code.

## Key Behaviors

1. **Message-Driven Processing**: All data flows through the rule engine as TbMsg messages, triggered by device telemetry, attributes, events, or API calls.

2. **Visual Programming Model**: Rule chains are designed in a drag-and-drop UI, connecting nodes with relation types (Success, Failure, True, False, etc.).

3. **Actor-Based Execution**: Each rule chain and rule node runs as an actor, providing isolation, concurrency, and fault tolerance.

4. **Pluggable Node Architecture**: Over 60 built-in node types across 6 categories, with support for custom node development.

5. **Nested Rule Chains**: Rule chains can invoke other rule chains, enabling modular, reusable processing logic.

6. **Multi-Tenant Isolation**: Each tenant has independent rule chains with configurable queue isolation.

## Architecture

```mermaid
graph TB
    subgraph "Message Sources"
        DEV[Device Messages]
        API[API Calls]
        ALARM[Alarm Events]
        SCHED[Scheduled Tasks]
    end

    subgraph "Rule Engine"
        Q[Message Queue]
        RCA[Rule Chain Actor]

        subgraph "Rule Chain"
            N1[Filter Node]
            N2[Enrichment Node]
            N3[Transform Node]
            N4[Action Node]
            N5[External Node]
        end
    end

    subgraph "Outputs"
        DB[(Database)]
        NOTIFY[Notifications]
        EXT[External Systems]
    end

    DEV --> Q
    API --> Q
    ALARM --> Q
    SCHED --> Q

    Q --> RCA
    RCA --> N1
    N1 -->|True| N2
    N1 -->|False| N4
    N2 --> N3
    N3 --> N4
    N4 --> N5

    N4 --> DB
    N4 --> NOTIFY
    N5 --> EXT
```

## Core Concepts

### Rule Chain

A rule chain is a container for rule nodes and their connections. It defines a message processing pipeline.

| Property | Type | Description |
|----------|------|-------------|
| id | RuleChainId | Unique identifier |
| tenantId | TenantId | Owning tenant |
| name | String | User-defined name |
| type | RuleChainType | CORE or EDGE |
| firstRuleNodeId | RuleNodeId | Entry point node |
| root | Boolean | Default chain for tenant |
| debugMode | Boolean | Enable debug logging |

### Rule Node

A rule node is a single processing step within a rule chain.

| Property | Type | Description |
|----------|------|-------------|
| id | RuleNodeId | Unique identifier |
| ruleChainId | RuleChainId | Parent chain |
| type | String | Fully qualified class name |
| name | String | User-defined name |
| configuration | JsonNode | Node-specific settings |
| debugMode | Boolean | Debug this node |

### TbMsg (Message)

TbMsg is the immutable message that flows through rule chains.

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Unique message ID |
| ts | long | Timestamp (milliseconds) |
| type | String | Message type (e.g., "Post telemetry") |
| originator | EntityId | Source entity (device, asset, etc.) |
| customerId | CustomerId | Associated customer |
| metaData | TbMsgMetaData | Key-value metadata |
| data | String | JSON payload |
| ruleChainId | RuleChainId | Current processing chain |
| ruleNodeId | RuleNodeId | Current processing node |

### Node Connections

Nodes connect via typed relations that determine message routing:

```mermaid
graph LR
    FILTER[Filter Node] -->|True| SAVE[Save Telemetry]
    FILTER -->|False| LOG[Log Node]
    FILTER -->|Failure| ALERT[Create Alarm]

    SAVE -->|Success| DONE[End]
    SAVE -->|Failure| ALERT
```

## Message Types

Messages entering the rule engine have specific types:

### Device Messages

| Type | Trigger | Data Content |
|------|---------|--------------|
| POST_TELEMETRY_REQUEST | Device sends telemetry | Telemetry key-values |
| POST_ATTRIBUTES_REQUEST | Device sends attributes | Attribute key-values |
| TO_SERVER_RPC_REQUEST | Device initiates RPC | Method and params |
| ACTIVITY_EVENT | Device becomes active | Activity metadata |
| INACTIVITY_EVENT | Device becomes inactive | Inactivity metadata |
| CONNECT_EVENT | Device connects | Connection info |
| DISCONNECT_EVENT | Device disconnects | Disconnection info |

### Entity Lifecycle Messages

| Type | Trigger | Data Content |
|------|---------|--------------|
| ENTITY_CREATED | Entity created | Entity data |
| ENTITY_UPDATED | Entity modified | Updated entity |
| ENTITY_DELETED | Entity removed | Deleted entity ID |
| ENTITY_ASSIGNED | Entity assigned to customer | Assignment info |
| ENTITY_UNASSIGNED | Entity unassigned | Unassignment info |

### Attribute Messages

| Type | Trigger | Data Content |
|------|---------|--------------|
| ATTRIBUTES_UPDATED | Attributes changed | Updated attributes |
| ATTRIBUTES_DELETED | Attributes removed | Deleted keys |

### Alarm Messages

| Type | Trigger | Data Content |
|------|---------|--------------|
| ALARM_CREATED | New alarm | Alarm data |
| ALARM_UPDATED | Alarm modified | Updated alarm |
| ALARM_SEVERITY_UPDATED | Severity changed | New severity |
| ALARM_ACK | Alarm acknowledged | Ack info |
| ALARM_CLEAR | Alarm cleared | Clear info |

### RPC Messages

| Type | Trigger | Data Content |
|------|---------|--------------|
| RPC_CALL_FROM_SERVER_TO_DEVICE | Server sends RPC | RPC request |
| RPC_QUEUED | RPC queued | Queue info |
| RPC_DELIVERED | RPC delivered | Delivery confirmation |
| RPC_SUCCESSFUL | RPC completed | Response data |
| RPC_TIMEOUT | RPC timed out | Timeout info |
| RPC_FAILED | RPC failed | Error details |

## Node Categories

```mermaid
graph TB
    subgraph "FILTER"
        F1[Script Filter]
        F2[Message Type Filter]
        F3[Originator Type Filter]
        F4[Check Relation]
    end

    subgraph "ENRICHMENT"
        E1[Originator Attributes]
        E2[Related Attributes]
        E3[Tenant Attributes]
        E4[Customer Attributes]
    end

    subgraph "TRANSFORMATION"
        T1[Script Transform]
        T2[Change Originator]
        T3[To Email]
        T4[Duplicate to Group]
    end

    subgraph "ACTION"
        A1[Save Telemetry]
        A2[Save Attributes]
        A3[Create Alarm]
        A4[RPC Call]
        A5[Log]
    end

    subgraph "EXTERNAL"
        X1[REST API Call]
        X2[Send Email]
        X3[Kafka]
        X4[AWS SNS/SQS]
    end

    subgraph "FLOW"
        FL1[Rule Chain Input]
        FL2[Rule Chain Output]
    end
```

### Category Summary

| Category | Purpose | Example Nodes |
|----------|---------|---------------|
| FILTER | Route based on conditions | Script Filter, Message Type Switch |
| ENRICHMENT | Add data to messages | Originator Attributes, Customer Attributes |
| TRANSFORMATION | Modify message content | Script Transform, Change Originator |
| ACTION | Perform operations | Save Telemetry, Create Alarm, RPC Call |
| EXTERNAL | Integrate external systems | REST API, Kafka, AWS, Email |
| FLOW | Control rule chain flow | Rule Chain Input, Rule Chain Output |

## Message Flow

### Basic Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant Q as Queue
    participant RCA as RuleChainActor
    participant N1 as FilterNode
    participant N2 as ActionNode
    participant DB as Database

    D->>Q: Post telemetry
    Q->>RCA: QueueToRuleEngineMsg
    RCA->>RCA: Get first node

    RCA->>N1: RuleChainToRuleNodeMsg
    N1->>N1: Evaluate condition
    N1->>RCA: tellNext(msg, "True")

    RCA->>RCA: Lookup connections for "True"
    RCA->>N2: RuleChainToRuleNodeMsg
    N2->>DB: Save telemetry
    N2->>RCA: tellSuccess(msg)

    RCA->>Q: Acknowledge message
```

### Nested Rule Chain Flow

```mermaid
sequenceDiagram
    participant RC1 as Parent Chain
    participant INPUT as Input Node
    participant RC2 as Child Chain
    participant OUTPUT as Output Node

    RC1->>INPUT: Message arrives
    INPUT->>INPUT: Push to stack
    INPUT->>RC2: ctx.input(msg, childChainId)

    RC2->>RC2: Process through nodes
    RC2->>OUTPUT: Reach output node

    OUTPUT->>OUTPUT: Pop from stack
    OUTPUT->>RC1: ctx.output(msg, relationType)
    RC1->>RC1: Continue from Input node
```

## Actor Model

### Actor Hierarchy

```mermaid
graph TB
    APP[AppActor] --> TA1[TenantActor 1]
    APP --> TA2[TenantActor 2]

    TA1 --> RCA1[RuleChainActor: Root]
    TA1 --> RCA2[RuleChainActor: Alarms]
    TA1 --> DA[DeviceActor]

    RCA1 --> RNA1[RuleNodeActor: Filter]
    RCA1 --> RNA2[RuleNodeActor: Save]
    RCA1 --> RNA3[RuleNodeActor: Alarm]

    RCA2 --> RNA4[RuleNodeActor: Process]
    RCA2 --> RNA5[RuleNodeActor: Notify]
```

### Rule Chain Actor

The RuleChainActor manages a single rule chain:

**Responsibilities:**
- Create and manage child RuleNodeActors
- Route messages between nodes based on relation types
- Handle nested rule chain invocations
- Process lifecycle events (create, update, delete)

**Message Types Handled:**
- QUEUE_TO_RULE_ENGINE_MSG - External messages entering the chain
- RULE_TO_RULE_CHAIN_TELL_NEXT_MSG - Node completed, route to next
- RULE_CHAIN_INPUT_MSG - Message from nested chain
- RULE_CHAIN_OUTPUT_MSG - Return to parent chain
- COMPONENT_LIFE_CYCLE_MSG - Chain updated/deleted

### Rule Node Actor

The RuleNodeActor wraps a single TbNode implementation:

**Responsibilities:**
- Instantiate TbNode via reflection
- Initialize node with configuration
- Route messages to node's `onMsg()` method
- Track execution count to prevent infinite loops
- Handle node errors gracefully

## Node Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Creating: Actor created
    Creating --> Initializing: Load configuration
    Initializing --> Active: init() succeeds
    Initializing --> Failed: init() throws

    Active --> Processing: Message received
    Processing --> Active: onMsg() completes
    Processing --> Active: tellNext/tellSuccess
    Processing --> Failed: Unhandled exception

    Active --> Updating: Configuration changed
    Updating --> Initializing: Recreate node

    Active --> Destroying: Chain deleted
    Failed --> Destroying: Recovery failed
    Destroying --> [*]: destroy() called
```

### TbNode Interface

```
interface TbNode {
    // Called once when node is created
    init(ctx: TbContext, config: TbNodeConfiguration): void

    // Called for each message
    onMsg(ctx: TbContext, msg: TbMsg): void

    // Called when node is removed
    destroy(): void

    // Called on cluster partition changes
    onPartitionChangeMsg(ctx: TbContext, msg: PartitionChangeMsg): void

    // Called during node version upgrades
    upgrade(fromVersion: int, oldConfig: JsonNode): Pair<Boolean, JsonNode>
}
```

## TbContext

The TbContext provides services and routing methods to rule nodes:

### Message Routing

| Method | Description |
|--------|-------------|
| `tellSuccess(msg)` | Route to SUCCESS connections |
| `tellNext(msg, relationType)` | Route to specific relation type |
| `tellNext(msg, relationTypes)` | Route to multiple relation types |
| `tellFailure(msg, exception)` | Route to FAILURE connections |
| `tellSelf(msg, delayMs)` | Schedule self-message |
| `ack(msg)` | Acknowledge without routing |

### Nested Chains

| Method | Description |
|--------|-------------|
| `input(msg, ruleChainId)` | Enter nested rule chain |
| `output(msg, relationType)` | Return to parent chain |

### Message Creation

| Method | Description |
|--------|-------------|
| `newMsg(...)` | Create new message |
| `transformMsg(...)` | Transform with new data |
| `transformMsgOriginator(...)` | Change originator |

### Service Access

TbContext provides access to 60+ platform services:
- Device, Asset, Customer services
- Telemetry and Attribute services
- Alarm service
- Notification services
- Relation service
- Cache services

## Connection Types

### Standard Connections

| Type | Usage |
|------|-------|
| Success | Default successful completion |
| Failure | Error or exception occurred |
| True | Filter condition passed |
| False | Filter condition failed |

### Node-Specific Connections

| Node | Custom Relations |
|------|------------------|
| Message Type Switch | One per message type |
| Originator Type Switch | One per entity type |
| Create Alarm | Created, Updated, Cleared |
| Device Profile | Alarm, RPC Response, etc. |

## @RuleNode Annotation

Rule nodes are declared using the `@RuleNode` annotation:

```
@RuleNode(
    type = ComponentType.FILTER,
    name = "script",
    nodeDescription = "Filter messages using JavaScript",
    nodeDetails = "Evaluates JS returning true/false",
    configClazz = TbJsFilterNodeConfiguration.class,
    relationTypes = {"True", "False"},
    clusteringMode = ComponentClusteringMode.ENABLED
)
```

| Attribute | Description |
|-----------|-------------|
| type | Category (FILTER, ACTION, etc.) |
| name | Display name |
| nodeDescription | Short description |
| nodeDetails | Detailed documentation |
| configClazz | Configuration class |
| relationTypes | Output connection types |
| clusteringMode | Singleton or per-partition |
| inEnabled | Can receive messages |
| outEnabled | Can send messages |
| ruleChainNode | Is a flow control node |

## Debug Mode

Debug mode captures message snapshots for troubleshooting:

```mermaid
graph LR
    MSG[Message] --> NODE[Rule Node]
    NODE --> DEBUG{Debug?}
    DEBUG -->|Yes| CAPTURE[Capture Snapshot]
    DEBUG -->|No| SKIP[Skip]
    CAPTURE --> STORE[(Debug Events)]
    NODE --> NEXT[Next Node]
```

**Captured Information:**
- Input message (type, data, metadata)
- Output message (if modified)
- Relation type used
- Processing time
- Error details (if failed)

## Configuration Example

### Simple Telemetry Processing Chain

```mermaid
graph LR
    INPUT[Input] --> FILTER[Temperature Filter]
    FILTER -->|True| ENRICH[Add Device Info]
    FILTER -->|False| LOG[Log]
    ENRICH --> SAVE[Save Telemetry]
    SAVE -->|Success| ALARM[Check Threshold]
    ALARM -->|Created| NOTIFY[Send Email]
    ALARM -->|False| END[End]
```

**Filter Node Configuration:**
```json
{
  "scriptLang": "TBEL",
  "script": "return msg.temperature > 0;"
}
```

**Alarm Node Configuration:**
```json
{
  "alarmType": "High Temperature",
  "severity": "WARNING",
  "condition": {
    "condition": [{
      "key": {"type": "TIME_SERIES", "key": "temperature"},
      "predicate": {"type": "NUMERIC", "operation": "GREATER", "value": 30}
    }]
  }
}
```

## Root Rule Chain

Each tenant has a root rule chain that processes all incoming device messages:

```mermaid
graph TB
    subgraph "Root Chain"
        INPUT[Device Message] --> SWITCH[Message Type Switch]
        SWITCH -->|Post telemetry| TEL[Telemetry Chain]
        SWITCH -->|Post attributes| ATTR[Attributes Chain]
        SWITCH -->|RPC Request| RPC[RPC Chain]
        SWITCH -->|Other| DEFAULT[Default Handler]
    end
```

**Root chain receives:**
- All device telemetry
- All device attributes
- Device connectivity events
- RPC requests

## Queue Processing Strategies

Rule engine queues support different submit and processing strategies for handling message flow.

### Submit Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| BURST | Submit all messages immediately | High throughput, best effort |
| BATCH | Group messages into batches | Balanced throughput/latency |
| SEQUENTIAL_BY_ORIGINATOR | Order by entity | Per-device ordering required |
| SEQUENTIAL_BY_TENANT | Order by tenant | Tenant-level ordering |
| SEQUENTIAL | Strict global order | Strict ordering (low throughput) |

### Processing Strategies

| Strategy | On Failure | Use Case |
|----------|------------|----------|
| SKIP_ALL_FAILURES | Log and continue | Non-critical data |
| SKIP_ALL_FAILURES_AND_TIMED_OUT | Skip failed and timed out | Best effort processing |
| RETRY_ALL | Retry all messages | Critical data requiring reprocessing |
| RETRY_FAILED | Retry only failures | Mixed criticality |
| RETRY_TIMED_OUT | Retry only timeouts | Network-sensitive |
| RETRY_FAILED_AND_TIMED_OUT | Retry both | Reliable delivery |

### Retry Mechanism

When retry is enabled, the strategy applies exponential backoff:

```mermaid
graph TB
    START[Process Pack] --> CHECK{All Success?}
    CHECK -->|Yes| COMMIT[Commit]
    CHECK -->|No| RETRY_CHECK{Retry Allowed?}

    RETRY_CHECK -->|retries > maxRetries| COMMIT
    RETRY_CHECK -->|failedPct > threshold| COMMIT

    RETRY_CHECK -->|Yes| PAUSE[Pause]
    PAUSE --> DOUBLE[Double pause time<br/>up to maxPause]
    DOUBLE --> REPROCESS[Reprocess selected]
    REPROCESS --> CHECK
```

**Retry Parameters:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| `retries` | Maximum retry attempts | 3 |
| `failurePercentage` | Abort if failures exceed this % | 0 (disabled) |
| `pauseBetweenRetries` | Initial pause (seconds) | 3 |
| `maxPauseBetweenRetries` | Maximum pause after backoff | 3 |

The `failurePercentage` threshold prevents infinite retries when a large portion of messages fail (e.g., database down).

### Queue Configuration Example

```yaml
queue:
  rule-engine:
    queues:
      - name: Main
        topic: tb_rule_engine.main
        partitions: 10
        consumer-per-partition: true
        submit-strategy:
          type: BURST
          batch-size: 1000
        processing-strategy:
          type: SKIP_ALL_FAILURES
          retries: 3
          failure-percentage: 0
          pause-between-retries: 3
          max-pause-between-retries: 3

      - name: HighPriority
        topic: tb_rule_engine.hp
        partitions: 4
        submit-strategy:
          type: SEQUENTIAL_BY_ORIGINATOR
        processing-strategy:
          type: RETRY_FAILED_AND_TIMED_OUT
          retries: 5
```

### Strategy Selection Guide

```mermaid
graph TB
    START[Choose Strategy]
    Q1{Need ordering?}
    Q2{Per entity or tenant?}
    Q3{Critical data?}
    Q4{Can retry?}

    START --> Q1
    Q1 -->|Yes| Q2
    Q1 -->|No| BURST[BURST]

    Q2 -->|Entity| SEQ_ORIG[SEQUENTIAL_BY_ORIGINATOR]
    Q2 -->|Tenant| SEQ_TENANT[SEQUENTIAL_BY_TENANT]

    BURST --> Q3
    SEQ_ORIG --> Q3
    SEQ_TENANT --> Q3

    Q3 -->|Yes| Q4
    Q3 -->|No| SKIP[SKIP_ALL_FAILURES]

    Q4 -->|Yes| RETRY[RETRY_FAILED_AND_TIMED_OUT]
    Q4 -->|No| SKIP
```

## Performance Considerations

### Message Processing Limits

| Limit | Purpose | Configurable |
|-------|---------|--------------|
| Max rule node executions per message | Prevent infinite loops | Yes |
| Message queue timeout | Prevent stuck messages | Yes |
| Debug event retention | Limit storage | Yes |

### Best Practices

1. **Keep chains simple** - Break complex logic into nested chains
2. **Use filters early** - Discard irrelevant messages quickly
3. **Batch external calls** - Use batch nodes for external systems
4. **Monitor debug carefully** - Debug mode impacts performance
5. **Test with representative load** - Validate before production

## See Also

- [Rule Chain Structure](./rule-chain-structure.md) - Detailed chain composition
- [Message Flow (TbMsg)](./message-flow.md) - Message routing details
- [Node Categories](./node-categories.md) - All built-in nodes
- [Node Development Contract](./node-development-contract.md) - Custom node development
- [Actor System](../03-actor-system/README.md) - Underlying actor model
