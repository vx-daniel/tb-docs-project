# Transport Layer Contract

## Overview

The transport layer is the entry point for device communication. It handles protocol-specific details (MQTT, CoAP, HTTP, LwM2M, SNMP) while exposing a unified interface to the rest of the platform. All transports authenticate devices, convert protocol messages to internal format, manage sessions, and enforce rate limits.

## Key Behaviors

1. **Protocol Translation**: Converts protocol-specific messages (MQTT PUBLISH, CoAP POST, HTTP request) into unified internal messages.

2. **Device Authentication**: Validates credentials before accepting data from devices.

3. **Session Management**: Tracks active device connections for bidirectional communication.

4. **Rate Limiting**: Enforces per-device and per-tenant limits to prevent abuse.

5. **Asynchronous Processing**: All operations are non-blocking with callback-based responses.

6. **Gateway Support**: Special handling for gateway devices that manage multiple child devices.

## Architecture

```mermaid
graph TB
    subgraph "Protocol Layer"
        MQTT[MQTT Handler]
        HTTP[HTTP Handler]
        COAP[CoAP Handler]
        LWM2M[LwM2M Handler]
        SNMP[SNMP Handler]
    end

    subgraph "Transport API"
        TS[TransportService]
        TC[TransportContext]
        RL[RateLimitService]
        AUTH[Auth Validation]
    end

    subgraph "Core Platform"
        Q[Message Queue]
        DA[Device Actor]
    end

    MQTT --> TS
    HTTP --> TS
    COAP --> TS
    LWM2M --> TS
    SNMP --> TS

    TS --> AUTH
    TS --> RL
    TS --> Q
    Q --> DA
```

## Transport Types

| Type | Protocol | Description |
|------|----------|-------------|
| DEFAULT | HTTP, MQTT, CoAP | Standard handlers, no protocol-specific config |
| MQTT | MQTT 3.1.1/5.0 | Custom MQTT topics and payload formats |
| COAP | CoAP | Constrained device settings |
| LWM2M | LwM2M | Object model mapping for device management |
| SNMP | SNMP v2c/v3 | OID mapping for network devices |

## Core Interfaces

### TransportService

The central service that all protocol handlers use to communicate with the platform.

**Credential Validation:**

| Method | Credential Type | Description |
|--------|-----------------|-------------|
| process(ValidateDeviceTokenRequestMsg) | Access Token | Simple token authentication |
| process(ValidateBasicMqttCredRequestMsg) | MQTT Basic | Username/password for MQTT |
| process(ValidateDeviceX509CertRequestMsg) | X.509 Certificate | Certificate fingerprint validation |
| process(ValidateDeviceLwM2MCredentialsRequestMsg) | LwM2M | Protocol-specific credentials |

**Data Operations:**

| Method | Purpose |
|--------|---------|
| process(PostTelemetryMsg) | Device sends telemetry data |
| process(PostAttributeMsg) | Device sends client attributes |
| process(GetAttributeRequestMsg) | Device requests attributes |
| process(SubscribeToAttributeUpdatesMsg) | Subscribe to shared attribute changes |
| process(SubscribeToRPCMsg) | Subscribe to server-initiated RPC |
| process(ToDeviceRpcResponseMsg) | Device responds to RPC request |
| process(ToServerRpcRequestMsg) | Device initiates RPC to server |
| process(ClaimDeviceMsg) | Device claim request |

**Session Management:**

| Method | Purpose |
|--------|---------|
| registerAsyncSession | Register persistent connection (MQTT, CoAP observe) |
| registerSyncSession | Register request/response session (HTTP) |
| deregisterSession | Clean up session on disconnect |
| recordActivity | Update last activity timestamp |
| hasSession | Check if session is active |

**Device Operations:**

| Method | Purpose |
|--------|---------|
| process(GetOrCreateDeviceFromGatewayRequestMsg) | Gateway creates child device |
| process(ProvisionDeviceRequestMsg) | New device provisioning |
| getDevice | Retrieve device information |
| getDeviceCredentials | Retrieve device credentials |
| getEntityProfile | Get device/tenant profile |

### TransportContext

Base context providing shared services to all transport implementations.

| Component | Purpose |
|-----------|---------|
| transportService | Access to core transport operations |
| rateLimitService | Rate limiting enforcement |
| otaPackageDataCache | OTA firmware/software cache |
| transportResourceCache | Shared resource caching |
| executor | Thread pool for async operations |
| scheduler | Scheduled task execution |

### SessionMsgListener

Callback interface for receiving messages directed to a device session.

| Callback | When Triggered |
|----------|----------------|
| onGetAttributesResponse | Response to attribute request |
| onAttributeUpdate | Shared attribute changed |
| onToDeviceRpcRequest | Server sends RPC to device |
| onToServerRpcResponse | Response to device-initiated RPC |
| onRemoteSessionCloseCommand | Force disconnect from server |
| onDeviceDeleted | Device was deleted |
| onDeviceProfileUpdate | Device profile changed |
| onDeviceUpdate | Device metadata changed |
| onToTransportUpdateCredentials | Credentials were changed |

## Authentication Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant T as Transport Handler
    participant TS as TransportService
    participant CS as Credential Store
    participant DP as Device Profile

    D->>T: Connect with credentials
    T->>TS: ValidateDeviceTokenRequestMsg
    TS->>CS: Lookup by credential ID

    alt Valid credentials
        CS-->>TS: Device found
        TS->>DP: Get device profile
        DP-->>TS: Profile config
        TS-->>T: ValidateDeviceCredentialsResponse
        T->>T: Create session
        T->>TS: registerAsyncSession
        T-->>D: Connection accepted
    else Invalid credentials
        CS-->>TS: Not found
        TS-->>T: Validation failed
        T-->>D: Connection rejected
    end
```

## Session Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Connecting: Device connects
    Connecting --> Authenticating: Parse credentials
    Authenticating --> Active: Credentials valid
    Authenticating --> [*]: Credentials invalid

    Active --> Active: Send/receive data
    Active --> Active: Record activity
    Active --> Idle: No activity

    Idle --> Active: New message
    Idle --> Closing: Timeout

    Active --> Closing: Disconnect
    Active --> Closing: Credentials changed
    Active --> Closing: Device deleted

    Closing --> [*]: Cleanup complete
```

### Session Info

Each session is identified by SessionInfoProto containing:

| Field | Type | Description |
|-------|------|-------------|
| sessionIdMSB | long | Session UUID (most significant bits) |
| sessionIdLSB | long | Session UUID (least significant bits) |
| tenantIdMSB | long | Tenant UUID |
| tenantIdLSB | long | Tenant UUID |
| deviceIdMSB | long | Device UUID |
| deviceIdLSB | long | Device UUID |
| deviceName | string | Device name |
| deviceType | string | Device type (from profile) |
| deviceProfileIdMSB | long | Profile UUID |
| deviceProfileIdLSB | long | Profile UUID |
| customerIdMSB | long | Customer UUID (optional) |
| customerIdLSB | long | Customer UUID (optional) |
| gwSessionIdMSB | long | Gateway session (if child device) |
| gwSessionIdLSB | long | Gateway session (if child device) |

### Session Metadata

Each registered session tracks additional state:

| Field | Purpose |
|-------|---------|
| sessionType | ASYNC (persistent) or SYNC (request/response) |
| listener | Callback handler for incoming messages |
| scheduledFuture | Timeout task for sync sessions |
| subscribedToAttributes | Attribute update subscription state |
| subscribedToRPC | RPC subscription state |
| overwriteActivityTime | Gateway-specific activity override flag |

## Activity Tracking

### Reporting Strategies

The transport layer reports session activity using configurable strategies:

| Strategy | Behavior |
|----------|----------|
| ALL | Report every activity event |
| FIRST | Report only first activity per interval |
| LAST | Report only last activity per interval (default) |
| FIRST_AND_LAST | Report both first and last activity |

**Configuration:**
- `transport.activity.reporting_strategy`: Strategy type (default: LAST)
- `transport.sessions.report_timeout`: Reporting interval (default: 3000ms)
- `transport.sessions.inactivity_timeout`: Session expiration (default: 600000ms)

### Activity Reporting Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant T as Transport
    participant AM as ActivityManager
    participant Q as Queue

    D->>T: Send message
    T->>T: recordActivity(sessionInfo)
    T->>AM: Update activity state

    Note over AM: Report interval timer fires

    AM->>AM: Check strategy
    alt LAST strategy
        AM->>Q: Report last activity time
    else FIRST strategy
        AM->>Q: Report first activity only
    end

    Note over AM: Inactivity timeout

    AM->>AM: Session expired
    AM->>T: onStateExpiry(sessionId)
    T->>T: Close session
    T->>D: Session timeout notification
```

### Gateway Activity Override

For child devices connected through a gateway, activity time can be overwritten:

```mermaid
graph TB
    GW[Gateway Session] -->|owns| CD1[Child Device 1]
    GW -->|owns| CD2[Child Device 2]
    GW -->|owns| CD3[Child Device 3]

    CD1 -->|activity| AGG[Aggregate to Gateway]
    CD2 -->|activity| AGG
    CD3 -->|activity| AGG

    AGG -->|overwriteActivityTime=true| GW
```

When `overwriteActivityTime` is enabled (default for gateway sessions):
- Child device activity extends gateway session lifetime
- Gateway session uses `max(gateway_activity, child_activity)`
- Prevents gateway timeout while children are active

## Message Processing Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant T as Transport Handler
    participant RL as Rate Limiter
    participant TS as TransportService
    participant Q as Message Queue
    participant CB as Callback

    D->>T: Send telemetry
    T->>T: Parse protocol message
    T->>RL: Check rate limits

    alt Rate limit OK
        RL-->>T: Allowed
        T->>TS: process(PostTelemetryMsg, callback)
        TS->>Q: Publish to queue
        Q-->>TS: Ack
        TS->>CB: onSuccess()
        CB-->>T: Complete
        T-->>D: Acknowledge
    else Rate limited
        RL-->>T: Rejected
        T-->>D: Error: rate limited
    end
```

## Rate Limiting

### Limit Types

| Limit | Scope | Description |
|-------|-------|-------------|
| Device Messages | Per device | Messages per time window |
| Device Data Points | Per device | Telemetry data points per window |
| Tenant Messages | Per tenant | Aggregate messages across all devices |
| Tenant Data Points | Per tenant | Aggregate data points |
| Transport Messages | Per transport | Global transport-level limit |
| Gateway Messages | Per gateway | Messages from gateway device itself |
| Gateway Device Messages | Per child | Messages per child device through gateway |

### Rate Limit Hierarchy

The rate limit service uses a multi-level hierarchy with separate buckets:

```mermaid
graph TB
    MSG[Incoming Message] --> IP{IP Limit?}
    IP -->|Blocked| REJECT[Reject]
    IP -->|OK| TENANT{Tenant Enabled?}
    TENANT -->|Disabled| REJECT
    TENANT -->|OK| TLIMIT{Tenant Msg Limit?}
    TLIMIT -->|Exceeded| REJECT
    TLIMIT -->|OK| GW{Is Gateway?}

    GW -->|Yes| GWLIMIT{Gateway Limit?}
    GW -->|No| DEVLIMIT{Device Limit?}

    GWLIMIT -->|Exceeded| REJECT
    GWLIMIT -->|OK| CHILD{Child Device?}
    CHILD -->|Yes| GWDEVLIMIT{Gateway-Device Limit?}
    CHILD -->|No| PROCESS[Process Message]
    GWDEVLIMIT -->|Exceeded| REJECT
    GWDEVLIMIT -->|OK| PROCESS

    DEVLIMIT -->|Exceeded| REJECT
    DEVLIMIT -->|OK| PROCESS
```

### Entity Rate Limits Structure

Each entity (tenant, device, gateway) has three independent rate limit buckets:

| Bucket | Measures | Typical Config |
|--------|----------|----------------|
| Regular Messages | Message count | 100:1,1000:60 |
| Telemetry Messages | Telemetry message count | 100:1,1000:60 |
| Telemetry Data Points | Individual data points | 1000:1,10000:60 |

Configuration format: `count:seconds` where multiple entries are comma-separated for burst and sustained limits.

### IP-Based Rate Limiting

Protects against credential brute-force attacks:

| Configuration | Default | Description |
|---------------|---------|-------------|
| ip_limits_enabled | false | Enable IP tracking |
| max_wrong_credentials_per_ip | 10 | Failed auths before block |
| ip_block_timeout | 60000ms | Block duration |

**Flow:**
1. Track failed authentication attempts per IP address
2. After threshold exceeded, block all requests from IP
3. After timeout, remove from blocklist
4. On successful auth, reset counter for IP

### Entity Limits Cache

Prevents repeated max-entity limit checks:

```mermaid
sequenceDiagram
    participant GW as Gateway
    participant TS as TransportService
    participant ELC as EntityLimitsCache
    participant DS as DeviceService

    GW->>TS: Create child device "DeviceX"
    TS->>ELC: Check limit for tenant

    alt Already at limit (cached)
        ELC-->>TS: Limit reached (cached)
        TS-->>GW: Error: max devices reached
    else Not cached
        TS->>DS: GetOrCreateDevice
        alt Device created
            DS-->>TS: Success
        else Entity limit error
            DS-->>TS: ENTITY_LIMIT error
            TS->>ELC: Cache limit reached
        end
    end
```

## Gateway Protocol

Gateways require special handling for managing multiple child devices.

### Gateway Operations

| Operation | Purpose |
|-----------|---------|
| GetOrCreateDeviceFromGatewayRequest | Create/lookup child device by name |
| Connect event for child | Notify platform of child device activity |
| Disconnect event for child | Notify platform child went offline |
| Telemetry for child | Post data on behalf of child device |
| Attributes for child | Post attributes on behalf of child device |

### Gateway Flow

```mermaid
sequenceDiagram
    participant GW as Gateway
    participant T as Transport
    participant TS as TransportService
    participant DS as Device Service

    Note over GW,DS: Gateway connects and authenticates normally

    GW->>T: Telemetry for "Child Device A"
    T->>TS: GetOrCreateDeviceFromGatewayRequest
    TS->>DS: Find device by name in tenant

    alt Device exists
        DS-->>TS: Existing device
    else Device not found
        DS->>DS: Create new device
        DS->>DS: Create "Contains" relation to gateway
        DS-->>TS: New device
    end

    TS-->>T: Device info + session
    T->>TS: PostTelemetryMsg (child session)
    TS-->>T: Success
    T-->>GW: Acknowledge
```

## Transport Device Info

After successful authentication, the transport receives device information:

| Field | Description |
|-------|-------------|
| tenantId | Owning tenant |
| customerId | Assigned customer (if any) |
| deviceProfileId | Device profile reference |
| deviceId | Unique device identifier |
| deviceName | Human-readable name |
| deviceType | Type from profile |
| powerMode | Power saving mode (for LwM2M) |
| gateway | True if gateway device |
| additionalInfo | Extra metadata |

## Protocol Implementation Checklist

When implementing a new transport protocol:

### Required Operations

- [ ] **Authentication**: Validate credentials via TransportService
- [ ] **Session Registration**: Register with registerAsyncSession or registerSyncSession
- [ ] **Session Deregistration**: Clean up on disconnect
- [ ] **Telemetry Processing**: Convert to PostTelemetryMsg
- [ ] **Attribute Processing**: Convert to PostAttributeMsg
- [ ] **Activity Recording**: Call recordActivity on messages

### Optional Operations

- [ ] **Attribute Subscription**: Implement SessionMsgListener.onAttributeUpdate
- [ ] **RPC Handling**: Implement onToDeviceRpcRequest
- [ ] **Server RPC**: Support ToServerRpcRequestMsg
- [ ] **OTA Updates**: Support firmware/software download
- [ ] **Gateway Mode**: Support child device management

### Error Handling

- [ ] **Invalid Credentials**: Return appropriate error code
- [ ] **Rate Limited**: Return throttling error
- [ ] **Malformed Message**: Log and reject
- [ ] **Internal Error**: Return server error, don't expose details

## Protocol-Specific Adaptors

Each protocol has an adaptor that converts protocol messages to internal format:

```mermaid
graph LR
    subgraph "MQTT"
        MP[MQTT PUBLISH] --> MA[MqttTransportAdaptor]
        MA --> TM[Internal TbMsg]
    end

    subgraph "CoAP"
        CP[CoAP POST] --> CA[CoapTransportAdaptor]
        CA --> TM
    end

    subgraph "HTTP"
        HP[HTTP POST] --> HA[HTTP Controller]
        HA --> TM
    end
```

### Adaptor Responsibilities

1. **Parse payload** (JSON, Protobuf, CBOR)
2. **Extract timestamp** (or use server time)
3. **Validate structure** (required fields present)
4. **Convert to internal format** (TbMsg or Protobuf)
5. **Handle protocol errors** (malformed data)

## Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| UNAUTHORIZED | Invalid credentials | Check token/certificate |
| RATE_LIMITED | Too many requests | Slow down, implement backoff |
| BAD_REQUEST | Malformed message | Fix payload format |
| NOT_FOUND | Device/resource missing | Verify device exists |
| INTERNAL_ERROR | Server failure | Retry with backoff |

## Transport Caching

### Cache Types

| Cache | Purpose | Scope |
|-------|---------|-------|
| DeviceProfileCache | Device profile configuration | Per transport instance |
| TenantProfileCache | Tenant-level settings | Per transport instance |
| TransportResourceCache | LwM2M models, custom resources | Shared |
| OtaPackageDataCache | Firmware/software packages | Shared |
| EntityLimitsCache | Max entity limit checks | Per tenant |

### Cache Update Flow

Caches are updated via notification consumer:

```mermaid
sequenceDiagram
    participant CS as Core Service
    participant Q as Notification Queue
    participant TS as TransportService
    participant C as Cache

    CS->>Q: DeviceProfileUpdate notification
    Q->>TS: ToTransportMsg

    alt Session exists
        TS->>C: Update cache
        TS->>TS: Notify session listener
    else Broadcast update
        TS->>C: Update cache globally
    end
```

### Notification Message Types

| Message Type | Trigger | Action |
|--------------|---------|--------|
| EntityUpdateMsg | Device/profile changed | Update cache, notify sessions |
| EntityDeleteMsg | Device/profile deleted | Evict from cache, close sessions |
| ResourceUpdateMsg | LwM2M/custom resource changed | Update resource cache |
| ResourceDeleteMsg | Resource removed | Evict from resource cache |
| GetAttributesResponse | Attribute request completed | Deliver to session listener |
| AttributeUpdateNotification | Shared attribute changed | Deliver to subscribed sessions |
| ToDeviceRpcRequest | Server sends RPC | Deliver to session listener |
| SessionCloseNotification | Force disconnect | Close session |

## Thread Model

### Executor Pools

| Pool | Size | Purpose |
|------|------|---------|
| TransportContext.executor | 50 (work-stealing) | Async message processing |
| transportCallbackExecutor | 20 (work-stealing) | Callback handling |
| scheduler | Configurable | Session timeouts, activity reporting |

### Message Processing

```mermaid
graph TB
    subgraph "Inbound"
        IQ[Protocol Handler] -->|parse| WS[Work-Stealing Pool]
        WS -->|validate| RL[Rate Limiter]
        RL -->|queue| MQ[Message Queue]
    end

    subgraph "Outbound"
        NQ[Notification Consumer] -->|dispatch| CE[Callback Executor]
        CE -->|deliver| SL[Session Listener]
    end
```

## See Also

- [MQTT Protocol](./mqtt.md) - MQTT-specific implementation
- [Device Entity](../02-core-concepts/entities/device.md) - Device authentication
- [Telemetry](../02-core-concepts/data-model/telemetry.md) - Data format
- [Attributes](../02-core-concepts/data-model/attributes.md) - Attribute handling
- [Actor System](../03-actor-system/README.md) - Message processing
