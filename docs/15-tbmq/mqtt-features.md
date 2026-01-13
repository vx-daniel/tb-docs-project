# MQTT Protocol Features

## Overview

TBMQ provides full compliance with MQTT 3.1, 3.1.1, and 5.0 specifications. This document covers the MQTT protocol features supported by TBMQ, including Quality of Service levels, session management, retained messages, Last Will and Testament, shared subscriptions, and MQTT 5.0 specific features.

## Quality of Service (QoS)

MQTT defines three QoS levels that control message delivery guarantees:

### QoS Levels

| Level | Name | Guarantee | Use Case |
|-------|------|-----------|----------|
| 0 | At most once | Fire and forget | Non-critical telemetry |
| 1 | At least once | Delivery guaranteed (may duplicate) | Important messages |
| 2 | Exactly once | Single delivery guaranteed | Critical transactions |

### QoS 0 Flow

```mermaid
sequenceDiagram
    participant PUB as Publisher
    participant BROKER as TBMQ
    participant SUB as Subscriber

    PUB->>BROKER: PUBLISH (QoS 0)
    Note over BROKER: No acknowledgment
    BROKER->>SUB: PUBLISH (QoS 0)
    Note over SUB: No acknowledgment
```

### QoS 1 Flow

```mermaid
sequenceDiagram
    participant PUB as Publisher
    participant BROKER as TBMQ
    participant SUB as Subscriber

    PUB->>BROKER: PUBLISH (QoS 1)
    BROKER->>PUB: PUBACK
    BROKER->>SUB: PUBLISH (QoS 1)
    SUB->>BROKER: PUBACK
```

### QoS 2 Flow

```mermaid
sequenceDiagram
    participant PUB as Publisher
    participant BROKER as TBMQ
    participant SUB as Subscriber

    PUB->>BROKER: PUBLISH (QoS 2)
    BROKER->>PUB: PUBREC
    PUB->>BROKER: PUBREL
    BROKER->>PUB: PUBCOMP

    BROKER->>SUB: PUBLISH (QoS 2)
    SUB->>BROKER: PUBREC
    BROKER->>SUB: PUBREL
    SUB->>BROKER: PUBCOMP
```

### QoS Downgrade

The effective QoS is the minimum of publisher and subscriber QoS:

| Publisher QoS | Subscriber QoS | Effective QoS |
|---------------|----------------|---------------|
| 2 | 2 | 2 |
| 2 | 1 | 1 |
| 2 | 0 | 0 |
| 1 | 0 | 0 |

## Session Management

### Session Types

| Type | Clean Session Flag | Behavior |
|------|-------------------|----------|
| Clean | true (3.x) / Clean Start (5.0) | Session cleared on disconnect |
| Persistent | false | Session retained across disconnects |

### Persistent Session Storage

```mermaid
graph TB
    subgraph "Persistent Session Data"
        SUBS[Subscriptions]
        MSGS[Queued Messages]
        QOS[QoS 1/2 State]
    end

    DISCONNECT[Client Disconnects] --> STORE[Store in Redis]
    STORE --> SUBS
    STORE --> MSGS
    STORE --> QOS

    RECONNECT[Client Reconnects] --> RESTORE[Restore Session]
    RESTORE --> DELIVER[Deliver Queued Messages]
```

### Session Expiry (MQTT 5.0)

MQTT 5.0 introduces session expiry interval:

| Value | Behavior |
|-------|----------|
| 0 | Session expires immediately on disconnect |
| N seconds | Session expires N seconds after disconnect |
| 0xFFFFFFFF | Session never expires |

## Topics and Wildcards

### Topic Structure

Topics are hierarchical strings separated by `/`:

```
sensors/building1/floor2/temperature
devices/thermostat/commands
home/+/temperature
factory/#
```

### Wildcard Characters

| Wildcard | Description | Example | Matches |
|----------|-------------|---------|---------|
| `+` | Single level | `sensors/+/temp` | sensors/room1/temp, sensors/room2/temp |
| `#` | Multi level | `sensors/#` | sensors/a, sensors/a/b/c |

### Topic Matching Examples

| Subscription | Published Topic | Match? |
|--------------|-----------------|--------|
| `home/+/temperature` | `home/living/temperature` | Yes |
| `home/+/temperature` | `home/living/room1/temperature` | No |
| `home/#` | `home/living/temperature` | Yes |
| `home/#` | `home/living/room1/temperature` | Yes |
| `+/+/temperature` | `home/living/temperature` | Yes |

## Retained Messages

Retained messages are stored by the broker and delivered to new subscribers:

```mermaid
sequenceDiagram
    participant PUB as Publisher
    participant BROKER as TBMQ
    participant SUB1 as Subscriber 1
    participant SUB2 as Subscriber 2

    PUB->>BROKER: PUBLISH (retain=true)
    Note over BROKER: Store retained message
    BROKER->>SUB1: PUBLISH (already subscribed)

    Note over SUB2: Subscribes later
    SUB2->>BROKER: SUBSCRIBE
    BROKER->>SUB2: PUBLISH (retained message)
```

### Retained Message Behavior

| Action | Behavior |
|--------|----------|
| Publish with retain=true | Store/update retained message |
| Publish empty payload with retain=true | Delete retained message |
| Subscribe to topic with retained | Receive retained message immediately |

## Last Will and Testament (LWT)

LWT messages are published when a client disconnects unexpectedly:

```mermaid
sequenceDiagram
    participant CLIENT as Client
    participant BROKER as TBMQ
    participant OTHER as Other Clients

    CLIENT->>BROKER: CONNECT (with Will message)
    Note over BROKER: Store Will message

    Note over CLIENT: Connection lost unexpectedly
    BROKER->>OTHER: PUBLISH (Will message)
```

### LWT Configuration

| Field | Description |
|-------|-------------|
| Will Topic | Topic for Will message |
| Will Message | Payload to publish |
| Will QoS | QoS level for Will |
| Will Retain | Retain flag for Will |
| Will Delay (5.0) | Delay before publishing |

## Shared Subscriptions

Shared subscriptions distribute messages among multiple subscribers:

### Topic Format

```
$share/{ShareName}/{TopicFilter}
```

Example: `$share/group1/sensors/+/temperature`

### Load Balancing

```mermaid
graph TB
    PUB[Publisher] --> BROKER[TBMQ]
    BROKER --> SHARE["$share/group/topic"]
    SHARE -->|Round Robin| S1[Subscriber 1]
    SHARE -->|Round Robin| S2[Subscriber 2]
    SHARE -->|Round Robin| S3[Subscriber 3]
```

### Shared Subscription Behavior

| Scenario | Behavior |
|----------|----------|
| Normal operation | Messages distributed round-robin |
| Subscriber disconnects | Remaining subscribers share load |
| All subscribers disconnect | Messages queued (APPLICATION clients) |

### Device vs Application Clients

| Client Type | Storage | Shared Sub Behavior |
|-------------|---------|---------------------|
| DEVICE | Redis | Per-client queuing when offline |
| APPLICATION | Kafka | Consumer group distribution |

## MQTT 5.0 Features

### Reason Codes

MQTT 5.0 provides detailed reason codes for all acknowledgments:

| Code | Name | Description |
|------|------|-------------|
| 0x00 | Success | Operation successful |
| 0x80 | Unspecified error | Generic error |
| 0x81 | Malformed packet | Protocol error |
| 0x82 | Protocol error | Violation of protocol |
| 0x83 | Implementation specific | Broker-specific error |
| 0x87 | Not authorized | Permission denied |

### User Properties

Key-value metadata attached to packets:

```
PUBLISH (topic=sensors/temp)
  User Properties:
    - deviceId: sensor-001
    - location: building-a
    - priority: high
```

### Message Expiry

Messages can have an expiration time:

```mermaid
graph LR
    PUB[Publish with expiry=60s] --> BROKER[TBMQ]
    BROKER -->|< 60s| DELIVER[Deliver to subscriber]
    BROKER -->|> 60s| DISCARD[Discard expired message]
```

### Topic Alias

Reduces bandwidth by replacing topic names with numeric aliases:

```mermaid
sequenceDiagram
    participant C as Client
    participant B as TBMQ

    C->>B: PUBLISH topic="sensors/building1/floor2/room5/temperature" alias=1
    Note over B: Store alias mapping
    C->>B: PUBLISH alias=1 (no topic)
    Note over B: Resolve alias to full topic
```

### Flow Control

MQTT 5.0 allows clients to limit in-flight messages:

| Property | Purpose |
|----------|---------|
| Receive Maximum | Max concurrent QoS 1/2 messages |
| Maximum Packet Size | Limit packet size |
| Topic Alias Maximum | Max topic aliases |

### Request-Response Pattern

MQTT 5.0 supports request-response via response topics:

```mermaid
sequenceDiagram
    participant REQ as Requester
    participant BROKER as TBMQ
    participant RESP as Responder

    REQ->>BROKER: PUBLISH (topic=request/cmd, responseTopic=response/123)
    BROKER->>RESP: PUBLISH (topic=request/cmd, responseTopic=response/123)
    RESP->>BROKER: PUBLISH (topic=response/123)
    BROKER->>REQ: PUBLISH (topic=response/123)
```

### Subscription Options (MQTT 5.0)

| Option | Description |
|--------|-------------|
| No Local | Don't receive own publications |
| Retain As Published | Preserve retain flag |
| Retain Handling | Control retained message delivery |
| Maximum QoS | Limit subscription QoS |

## Keep Alive

Keep alive ensures connection health:

```mermaid
sequenceDiagram
    participant C as Client
    participant B as TBMQ

    C->>B: CONNECT (keepAlive=60)
    Note over C,B: Normal operation

    Note over C: No data to send
    C->>B: PINGREQ
    B->>C: PINGRESP

    Note over C: Keep alive timeout
    B->>B: Close connection
```

### Keep Alive Behavior

| Condition | Action |
|-----------|--------|
| Client sends PINGREQ | Broker responds PINGRESP |
| No activity for 1.5 × keepAlive | Broker closes connection |
| keepAlive = 0 | Keep alive disabled |

## MQTT over WebSocket

TBMQ supports MQTT over WebSocket for browser-based clients:

| Endpoint | Protocol | Port |
|----------|----------|------|
| ws://host:8084/mqtt | WebSocket | 8084 |
| wss://host:8085/mqtt | WebSocket Secure | 8085 |

## Best Practices

### QoS Selection

| Use Case | Recommended QoS |
|----------|-----------------|
| Telemetry (high frequency) | QoS 0 |
| Important sensor data | QoS 1 |
| Commands and confirmations | QoS 1 or 2 |
| Financial transactions | QoS 2 |

### Topic Design

| Practice | Example |
|----------|---------|
| Use hierarchy | `company/building/floor/room/sensor` |
| Be specific | `sensors/temperature` not `data` |
| Avoid deep nesting | Max 5-7 levels |
| Use consistent naming | camelCase or snake_case |

### Session Management

| Scenario | Recommendation |
|----------|----------------|
| Mobile apps | Persistent session with expiry |
| Backend services | Clean session |
| IoT devices | Persistent session |
| Dashboard | Clean session |

## See Also

- [TBMQ Architecture](./tbmq-architecture.md) - Broker internals
- [Transport Layer](../05-transport-layer/mqtt-transport.md) - ThingsBoard MQTT
- [Integrations](../14-integrations/messaging-integrations.md) - MQTT integration
