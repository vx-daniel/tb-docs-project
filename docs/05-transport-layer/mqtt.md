# MQTT Protocol

## Overview

MQTT (Message Queuing Telemetry Transport) is the primary device communication protocol in the platform. It provides a lightweight, publish/subscribe messaging model ideal for IoT devices with limited bandwidth and processing power. The platform supports MQTT 3.1.1 and MQTT 5.0, with features including QoS levels 0 and 1, persistent sessions, and bidirectional communication.

## Key Behaviors

1. **Access Token Authentication**: Devices authenticate using access tokens passed as the MQTT username.

2. **X.509 Certificate Authentication**: Devices can authenticate using client certificates for enhanced security.

3. **Two API Versions**: v1 topics use full paths (`v1/devices/me/telemetry`), v2 topics use short paths (`v2/t`) to reduce bandwidth.

4. **Multiple Payload Formats**: Supports JSON (default), Protocol Buffers, and device profile-specific formats.

5. **Bidirectional Communication**: Devices can publish data and subscribe to receive commands, attribute updates, and RPC requests.

6. **Gateway Support**: Gateway devices can manage multiple child devices through dedicated gateway topics.

7. **Sparkplug B Support**: Industrial IoT integration via the Sparkplug B specification.

## Connection Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant M as MQTT Handler
    participant TS as TransportService
    participant CS as Credential Store
    participant DP as Device Profile

    D->>M: CONNECT (username=accessToken)
    M->>TS: ValidateBasicMqttCredRequestMsg
    TS->>CS: Lookup credentials

    alt Valid Credentials
        CS-->>TS: Device info
        TS->>DP: Get device profile
        DP-->>TS: Profile configuration
        TS-->>M: ValidateDeviceCredentialsResponse
        M->>M: Initialize session
        M->>TS: registerAsyncSession
        M-->>D: CONNACK (Success)
    else Invalid Credentials
        CS-->>TS: Not found
        TS-->>M: Validation failed
        M-->>D: CONNACK (Not Authorized)
        M->>M: Close connection
    end
```

## Authentication Methods

### Access Token Authentication

The simplest authentication method. Device passes access token as MQTT username.

| MQTT Field | Value |
|------------|-------|
| Username | Device access token |
| Password | (optional) |
| Client ID | Any unique identifier |

**Example:**
```
Username: A1B2C3D4E5F6G7H8
Password:
ClientId: myDevice001
```

### MQTT Basic Credentials

Uses both username and password for authentication.

| MQTT Field | Value |
|------------|-------|
| Username | Configured username |
| Password | Configured password |
| Client ID | Configured client ID |

### X.509 Certificate Authentication

Devices authenticate using TLS client certificates. The platform validates the certificate fingerprint against registered credentials.

**Requirements:**
- TLS connection required
- Client certificate must be configured in device credentials
- Certificate validity checked (can be disabled via configuration)

## Topic Structure

### v1 Device Topics

Standard topics for individual device communication.

| Operation | Topic | Direction | Payload |
|-----------|-------|-----------|---------|
| Post telemetry | `v1/devices/me/telemetry` | Device → Server | JSON telemetry |
| Post attributes | `v1/devices/me/attributes` | Device → Server | JSON attributes |
| Request attributes | `v1/devices/me/attributes/request/{requestId}` | Device → Server | `{"clientKeys":"k1,k2","sharedKeys":"k3"}` |
| Attribute response | `v1/devices/me/attributes/response/{requestId}` | Server → Device | JSON attributes |
| Subscribe to attributes | `v1/devices/me/attributes` | Server → Device | JSON shared attributes |
| Server RPC request | `v1/devices/me/rpc/request/{requestId}` | Server → Device | `{"method":"name","params":{}}` |
| Server RPC response | `v1/devices/me/rpc/response/{requestId}` | Device → Server | JSON response |
| Client RPC request | `v1/devices/me/rpc/request/{requestId}` | Device → Server | `{"method":"name","params":{}}` |
| Device claim | `v1/devices/me/claim` | Device → Server | `{"secretKey":"...", "durationMs":...}` |

### v2 Short Topics

Bandwidth-efficient topics for constrained devices.

| Operation | Topic | Format Variants |
|-----------|-------|-----------------|
| Post telemetry | `v2/t` | `v2/t/j` (JSON), `v2/t/p` (Protobuf) |
| Post attributes | `v2/a` | `v2/a/j` (JSON), `v2/a/p` (Protobuf) |
| Request attributes | `v2/a/req/{requestId}` | `v2/a/req/j/{id}`, `v2/a/req/p/{id}` |
| Attribute response | `v2/a/res/{requestId}` | `v2/a/res/j/{id}`, `v2/a/res/p/{id}` |
| RPC request (to device) | `v2/r/req/{requestId}` | `v2/r/req/j/{id}`, `v2/r/req/p/{id}` |
| RPC response (from device) | `v2/r/res/{requestId}` | `v2/r/res/j/{id}`, `v2/r/res/p/{id}` |

### Gateway Topics

Topics for gateway devices managing child devices.

| Operation | Topic | Payload |
|-----------|-------|---------|
| Child connect | `v1/gateway/connect` | `{"device":"name"}` |
| Child disconnect | `v1/gateway/disconnect` | `{"device":"name"}` |
| Telemetry (multi-device) | `v1/gateway/telemetry` | `{"Device A":[{...}], "Device B":[{...}]}` |
| Attributes (multi-device) | `v1/gateway/attributes` | `{"Device A":{...}, "Device B":{...}}` |
| Attribute request | `v1/gateway/attributes/request` | `{"id":1, "device":"name", "client":true, "keys":["k1"]}` |
| Attribute response | `v1/gateway/attributes/response` | Response payload |
| RPC response | `v1/gateway/rpc` | `{"device":"name", "id":1, "data":{}}` |
| Device claim | `v1/gateway/claim` | `{"Device A":{"secretKey":"..."}}` |

### Provisioning Topics

For automatic device registration.

| Operation | Topic |
|-----------|-------|
| Provision request | `/provision/request` |
| Provision response | `/provision/response` |

### OTA Update Topics

For firmware and software updates.

| Operation | Topic |
|-----------|-------|
| Firmware request | `v2/fw/request/{requestId}/chunk/{chunk}` |
| Firmware response | `v2/fw/response/{requestId}/chunk/{chunk}` |
| Firmware error | `v2/fw/error` |
| Software request | `v2/sw/request/{requestId}/chunk/{chunk}` |
| Software response | `v2/sw/response/{requestId}/chunk/{chunk}` |
| Software error | `v2/sw/error` |

## Topic Flow Diagram

```mermaid
graph TB
    subgraph "Device Operations"
        PUB_TEL[Publish Telemetry] -->|v1/devices/me/telemetry| SERVER
        PUB_ATTR[Publish Attributes] -->|v1/devices/me/attributes| SERVER
        REQ_ATTR[Request Attributes] -->|v1/devices/me/attributes/request/+| SERVER
    end

    subgraph "Server Operations"
        SERVER -->|v1/devices/me/attributes| SUB_ATTR[Attribute Updates]
        SERVER -->|v1/devices/me/rpc/request/+| RPC_REQ[RPC Requests]
    end

    subgraph "Device Responses"
        RPC_REQ -->|Process| DEVICE
        DEVICE -->|v1/devices/me/rpc/response/+| RPC_RESP[RPC Response]
        RPC_RESP --> SERVER
    end
```

## Payload Formats

### Telemetry Payload

**Simple format (server assigns timestamp):**
```json
{
  "temperature": 25.5,
  "humidity": 60,
  "status": "active"
}
```

**With explicit timestamp:**
```json
{
  "ts": 1634567890123,
  "values": {
    "temperature": 25.5,
    "humidity": 60
  }
}
```

**Multiple timestamps:**
```json
[
  {"ts": 1634567890000, "values": {"temperature": 25.5}},
  {"ts": 1634567891000, "values": {"temperature": 25.7}}
]
```

### Attributes Payload

**Client attributes:**
```json
{
  "firmware_version": "1.2.3",
  "ip_address": "192.168.1.100",
  "hardware_model": "ESP32"
}
```

### RPC Request Payload

**Server to device:**
```json
{
  "method": "setConfiguration",
  "params": {
    "interval": 30,
    "enabled": true
  }
}
```

**Device response:**
```json
{
  "result": "success"
}
```

### Gateway Telemetry Payload

```json
{
  "Sensor A": [
    {"ts": 1634567890123, "values": {"temperature": 25.5}}
  ],
  "Sensor B": [
    {"ts": 1634567890456, "values": {"humidity": 60}}
  ]
}
```

## QoS Support

| QoS Level | Name | Behavior | Supported |
|-----------|------|----------|-----------|
| 0 | At most once | Fire and forget, no acknowledgment | Yes |
| 1 | At least once | Guaranteed delivery with PUBACK | Yes |
| 2 | Exactly once | Guaranteed single delivery | No |

**QoS Selection:**
- The platform caps requested QoS at level 1
- Requested QoS 2 is downgraded to QoS 1
- Server-to-device messages use the subscribed QoS

## Subscription Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant M as MQTT Handler
    participant TS as TransportService
    participant DA as Device Actor

    D->>M: SUBSCRIBE (v1/devices/me/rpc/request/+)
    M->>TS: SubscribeToRPCMsg
    TS->>DA: Register subscription
    DA-->>TS: Confirmed
    M-->>D: SUBACK (granted QoS)

    Note over D,DA: Later, server initiates RPC

    DA->>TS: RPC request
    TS->>M: ToDeviceRpcRequestMsg
    M-->>D: PUBLISH (v1/devices/me/rpc/request/123)
    D->>D: Process RPC
    D->>M: PUBLISH (v1/devices/me/rpc/response/123)
    M->>TS: ToDeviceRpcResponseMsg
    TS->>DA: Deliver response
```

## Message Processing

### Publish Message Handling

```mermaid
graph TB
    MSG[PUBLISH Message] --> PARSE[Parse Topic]

    PARSE --> GW{Gateway Topic?}
    GW -->|Yes| GWHANDLER[Gateway Handler]
    GW -->|No| SP{Sparkplug?}

    SP -->|Yes| SPHANDLER[Sparkplug Handler]
    SP -->|No| DEV[Device Handler]

    DEV --> TOPIC{Topic Type}
    TOPIC -->|telemetry| TEL[PostTelemetryMsg]
    TOPIC -->|attributes| ATTR[PostAttributeMsg]
    TOPIC -->|rpc/response| RPC[ToDeviceRpcResponseMsg]
    TOPIC -->|rpc/request| SRPC[ToServerRpcRequestMsg]
    TOPIC -->|claim| CLAIM[ClaimDeviceMsg]
    TOPIC -->|fw/request| OTA[OTA Package Request]

    TEL --> TS[Transport Service]
    ATTR --> TS
    RPC --> TS
    SRPC --> TS
    CLAIM --> TS
    OTA --> CACHE[OTA Package Cache]
```

### Gateway Message Flow

```mermaid
sequenceDiagram
    participant GW as Gateway
    participant M as MQTT Handler
    participant GH as Gateway Handler
    participant TS as TransportService
    participant DS as Device Service

    GW->>M: PUBLISH v1/gateway/connect
    M->>GH: onDeviceConnect
    GH->>TS: GetOrCreateDeviceFromGatewayRequest
    TS->>DS: Find/create device

    alt Device exists
        DS-->>TS: Existing device
    else Device not found
        DS->>DS: Create device
        DS->>DS: Create "Contains" relation
        DS-->>TS: New device
    end

    TS-->>GH: Device session info
    GH-->>M: Success
    M-->>GW: PUBACK

    Note over GW,DS: Gateway posts telemetry for child

    GW->>M: PUBLISH v1/gateway/telemetry
    M->>GH: onDeviceTelemetry
    GH->>GH: Parse device name from payload
    GH->>TS: PostTelemetryMsg (child session)
    TS-->>GH: Success
    M-->>GW: PUBACK
```

## Sparkplug B Support

The platform supports the Sparkplug B specification (v3.0.0) for industrial IoT integration.

### Sparkplug Topic Structure

```
spBv1.0/{group_id}/{message_type}/{edge_node_id}[/{device_id}]
```

| Message Type | Description | Direction |
|--------------|-------------|-----------|
| NBIRTH | Node birth certificate | Node → Server |
| NDEATH | Node death certificate | Node → Server |
| NDATA | Node data | Node → Server |
| NCMD | Node command | Server → Node |
| DBIRTH | Device birth certificate | Node → Server |
| DDEATH | Device death certificate | Node → Server |
| DDATA | Device data | Node → Server |
| DCMD | Device command | Server → Node |
| NRECORD | Node record message | Node → Server |
| DRECORD | Device record message | Node → Server |
| STATE | Critical application state | Bidirectional |

### Sparkplug Metric Features

**Metric Aliasing:**
- Birth certificates define metric name-to-alias mapping
- Subsequent DDATA/NDATA can use numeric aliases instead of names
- Reduces bandwidth for frequently reported metrics
- Platform tracks `nodeBirthMetrics` and `deviceBirthMetrics`

**Sequence Numbers:**
- `SPARKPLUG_SEQUENCE_NUMBER_KEY`: Message sequence tracking
- `SPARKPLUG_BD_SEQUENCE_NUMBER_KEY`: Birth/death sequence
- Enables detection of missed messages
- Validates message ordering

**Metric Data Types:**
Protocol Buffer serialization supporting all Sparkplug metric types including timestamps, quality codes, and metadata.

### Sparkplug Flow

```mermaid
sequenceDiagram
    participant EN as Edge Node
    participant M as MQTT Handler
    participant SPH as Sparkplug Handler
    participant TS as TransportService

    EN->>M: CONNECT
    M-->>EN: CONNACK

    EN->>M: SUBSCRIBE spBv1.0/+/NCMD/+
    M-->>EN: SUBACK

    EN->>M: PUBLISH spBv1.0/G1/NBIRTH/E1
    M->>SPH: onAttributesTelemetryProto (NBIRTH)
    SPH->>TS: Process node birth

    EN->>M: PUBLISH spBv1.0/G1/DBIRTH/E1/D1
    M->>SPH: onAttributesTelemetryProto (DBIRTH)
    SPH->>TS: Process device birth

    EN->>M: PUBLISH spBv1.0/G1/DDATA/E1/D1
    M->>SPH: onAttributesTelemetryProto (DDATA)
    SPH->>TS: Process device data
```

## Provisioning Flow

```mermaid
sequenceDiagram
    participant D as New Device
    participant M as MQTT Handler
    participant TS as TransportService
    participant PS as Provisioning Service

    D->>M: CONNECT (username="provision")
    M->>M: Set provision-only mode
    M-->>D: CONNACK

    D->>M: SUBSCRIBE /provision/response
    M-->>D: SUBACK

    D->>M: PUBLISH /provision/request
    Note right of D: {"deviceName":"...", "provisionDeviceKey":"...", "provisionDeviceSecret":"..."}
    M->>TS: ProvisionDeviceRequestMsg
    TS->>PS: Validate and provision

    alt Successful
        PS-->>TS: New credentials
        TS-->>M: ProvisionDeviceResponseMsg
        M-->>D: PUBLISH /provision/response
        Note right of D: {"credentialsType":"ACCESS_TOKEN", "accessToken":"..."}
        M->>M: Schedule disconnect (60s)
    else Failed
        PS-->>TS: Error
        TS-->>M: Error response
        M-->>D: PUBLISH /provision/response
        Note right of D: {"errorMsg":"...", "status":"FAILURE"}
    end
```

## OTA Updates Flow

```mermaid
sequenceDiagram
    participant D as Device
    participant M as MQTT Handler
    participant TS as TransportService
    participant CACHE as OTA Cache

    D->>M: SUBSCRIBE v2/fw/response/+/chunk/+
    M-->>D: SUBACK

    D->>M: PUBLISH v2/fw/request/1/chunk/0
    Note right of D: Payload: chunk size (e.g., "4096")
    M->>TS: GetOtaPackageRequestMsg
    TS-->>M: Package ID

    M->>CACHE: Get chunk 0
    CACHE-->>M: Chunk data
    M-->>D: PUBLISH v2/fw/response/1/chunk/0

    loop For each chunk
        D->>M: PUBLISH v2/fw/request/1/chunk/N
        M->>CACHE: Get chunk N
        CACHE-->>M: Chunk data
        M-->>D: PUBLISH v2/fw/response/1/chunk/N
    end
```

## Session Management

### Session States

```mermaid
stateDiagram-v2
    [*] --> Connecting: CONNECT received
    Connecting --> Authenticating: Parse credentials
    Authenticating --> ProvisionOnly: username="provision"
    Authenticating --> Connected: Valid credentials
    Authenticating --> [*]: Invalid credentials

    ProvisionOnly --> ProvisionOnly: Provisioning topics only
    ProvisionOnly --> [*]: Provisioned or timeout

    Connected --> Connected: PUBLISH/SUBSCRIBE
    Connected --> Connected: PINGREQ/PINGRESP
    Connected --> Disconnecting: DISCONNECT
    Connected --> Disconnecting: Error/Timeout

    Disconnecting --> [*]: Cleanup complete
```

### Keep-Alive

- Devices send PINGREQ to maintain connection
- Server responds with PINGRESP
- Gateway devices trigger ping for all child device sessions
- Activity recorded on each interaction

## Error Handling

### MQTT 5.0 Reason Codes

| Code | Name | When |
|------|------|------|
| 0x00 | SUCCESS | Operation completed |
| 0x80 | UNSPECIFIED_ERROR | Generic failure |
| 0x83 | IMPLEMENTATION_SPECIFIC_ERROR | Internal error |
| 0x87 | NOT_AUTHORIZED | Authentication failed |
| 0x8F | TOPIC_FILTER_INVALID | Invalid subscription topic |
| 0x99 | PAYLOAD_FORMAT_INVALID | Malformed payload |

### Error Responses

| Error Type | MQTT 3.1.1 Behavior | MQTT 5.0 Behavior |
|------------|---------------------|-------------------|
| Invalid payload | Close connection | PUBACK with error code |
| Invalid topic | Close connection | PUBACK with TOPIC_NAME_INVALID |
| Rate limited | Close connection | PUBACK with error code |
| Auth failure | CONNACK refused | CONNACK with reason code |

## Configuration Options

### Device Profile Settings

| Setting | Description | Default |
|---------|-------------|---------|
| transportPayloadType | JSON or PROTOBUF | JSON |
| sendAckOnValidationException | Send PUBACK on errors | false |
| telemetryTopic | Custom telemetry topic pattern | Standard |
| attributesTopic | Custom attributes topic pattern | Standard |

### Transport Settings (thingsboard.yml)

| Setting | Environment Variable | Default | Description |
|---------|---------------------|---------|-------------|
| bind_address | MQTT_BIND_ADDRESS | 0.0.0.0 | Bind address for MQTT listener |
| bind_port | MQTT_BIND_PORT | 1883 | Non-SSL MQTT port |
| timeout | MQTT_TIMEOUT | 10000 | Processing timeout (ms) |
| disconnectTimeout | MQTT_DISCONNECT_TIMEOUT | 1000 | Disconnect timeout (ms) |
| msgQueueSizePerDeviceLimit | MQTT_MSG_QUEUE_SIZE_PER_DEVICE_LIMIT | 100 | Per-device message queue size |

### Netty Configuration

| Setting | Environment Variable | Default | Description |
|---------|---------------------|---------|-------------|
| netty.max_payload_size | NETTY_MAX_PAYLOAD_SIZE | 65536 | Max MQTT payload (bytes) |
| netty.so_keep_alive | NETTY_SO_KEEPALIVE | false | TCP keep-alive |
| netty.leak_detector_level | NETTY_LEAK_DETECTOR_LEVEL | DISABLED | Resource leak detection |
| netty.boss_group_thread_count | NETTY_BOSS_GROUP_THREAD_COUNT | 1 | Netty boss threads |
| netty.worker_group_thread_count | NETTY_WORKER_GROUP_THREAD_COUNT | 12 | Netty worker threads |

**Network Pipeline:**
```mermaid
graph LR
    subgraph "Netty Pipeline"
        HAP[HAProxy Decoder] --> IPF[IP Filter]
        IPF --> SSL[SSL Handler]
        SSL --> DEC[MQTT Decoder]
        DEC --> ENC[MQTT Encoder]
        ENC --> MTH[MQTT Transport Handler]
    end
```

### HAProxy PROXY Protocol

| Setting | Environment Variable | Default | Description |
|---------|---------------------|---------|-------------|
| proxy_enabled | MQTT_PROXY_PROTOCOL_ENABLED | false | Enable HAProxy support |

When enabled:
- Adds HAProxyMessageDecoder to Netty pipeline
- Extracts real client IP from PROXY protocol header
- Uses ProxyIpFilter instead of IpFilter for rate limiting
- Required when load balancing MQTT through HAProxy

### Message Queue Management

Each device session has a bounded message queue:
- Default size: 100 messages per device
- Queue overflow: New messages dropped when full
- Thread-safe: ConcurrentLinkedQueue with ReentrantLock
- Atomic message ID sequencing for QoS tracking
- Snapshot capability for monitoring queue state

### MQTT 5.0 Configuration

| Feature | Description |
|---------|-------------|
| sessionExpiryInterval | Session expiry interval property |
| Reason codes | Detailed error codes in CONNACK, PUBACK, DISCONNECT |
| Graceful disconnect | DISCONNECT message with reason before close |
| Properties support | Full MQTT 5.0 properties via Netty MqttProperties |

**MQTT Version Detection:**
- Protocol level 3: MQTT 3.1
- Protocol level 4: MQTT 3.1.1 (most common)
- Protocol level 5: MQTT 5.0
- Automatic version detection from CONNECT message

**MQTT 5.0 Reason Codes:**

| Code | Name | Usage |
|------|------|-------|
| 0x00 | Success | Normal completion |
| 0x80 | Unspecified error | Generic failure |
| 0x82 | Protocol error | Protocol violation |
| 0x83 | Implementation specific | Internal error |
| 0x84 | Unsupported protocol version | Version mismatch |
| 0x85 | Client ID not valid | Invalid client identifier |
| 0x86 | Bad username/password | Credential error |
| 0x87 | Not authorized | Authorization failure |
| 0x8F | Topic filter invalid | Bad subscription |
| 0x97 | Quota exceeded | Rate limit hit |

**Graceful Disconnect:**
For MQTT 5.0 clients, the server sends a DISCONNECT message with reason code before closing the channel, allowing clients to handle disconnection gracefully.

### Gateway Configuration

| Setting | Environment Variable | Default | Description |
|---------|---------------------|---------|-------------|
| gatewayMetricsReportInterval | MQTT_GATEWAY_METRICS_REPORT_INTERVAL_SEC | 60 | Gateway metrics report interval (seconds) |

### Payload Adapters

ThingsBoard supports multiple payload adapters for different use cases:

| Adapter | Description |
|---------|-------------|
| JsonMqttAdaptor | Default JSON payload processing |
| ProtoMqttAdaptor | Protocol Buffers payload processing |
| BackwardCompatibilityAdaptor | Legacy payload format handling |

## Security Considerations

### TLS/SSL Configuration

| Setting | Environment Variable | Default | Description |
|---------|---------------------|---------|-------------|
| ssl.enabled | MQTT_SSL_ENABLED | false | Enable SSL/TLS |
| ssl.bind_address | MQTT_SSL_BIND_ADDRESS | 0.0.0.0 | SSL listener address |
| ssl.bind_port | MQTT_SSL_BIND_PORT | 8883 | SSL listener port |
| ssl.protocol | MQTT_SSL_PROTOCOL | TLSv1.2 | TLS protocol version |
| ssl.credentials.type | MQTT_SSL_CREDENTIALS_TYPE | PEM | PEM or KEYSTORE |

**PEM Credentials:**
| Setting | Description |
|---------|-------------|
| ssl.credentials.pem.cert_file | Server certificate file path |
| ssl.credentials.pem.key_file | Private key file path |
| ssl.credentials.pem.key_password | Private key password (optional) |

**Keystore Credentials:**
| Setting | Description |
|---------|-------------|
| ssl.credentials.keystore.type | JKS or PKCS12 |
| ssl.credentials.keystore.store_file | Keystore file path |
| ssl.credentials.keystore.store_password | Keystore password |
| ssl.credentials.keystore.key_alias | Key alias in keystore |
| ssl.credentials.keystore.key_password | Key password |

**X.509 Client Certificate Validation:**
- SHA3-256 hashing for certificate fingerprint validation
- Certificate chain verification supported
- Asynchronous validation with 10-second timeout
- `skip_validity_check_for_client_cert`: Skip expiration check (default: false)

### Access Control

- Devices can only access their own topics
- Gateway devices can access child device topics
- Provision-only sessions restricted to provisioning topics

### Rate Limiting

**Multi-Level Rate Limiting:**
- Per-device message limits
- Per-tenant aggregate limits
- Transport-level global limits
- Gateway-specific limits for child devices

**IP-Based Rate Limiting:**
| Setting | Default | Description |
|---------|---------|-------------|
| rate_limits.ip_limits_enabled | false | Enable IP tracking |
| rate_limits.max_wrong_credentials_per_ip | 10 | Failed auth threshold |
| rate_limits.ip_block_timeout | 60000ms | Block duration |

Protects against credential brute-force attacks by blocking IPs after repeated authentication failures.

## Implementation Example

### Device Publishing Telemetry

```
# Connect
CONNECT
  Client ID: device001
  Username: A1B2C3D4E5F6

# Publish telemetry
PUBLISH
  Topic: v1/devices/me/telemetry
  QoS: 1
  Payload: {"temperature": 25.5, "humidity": 60}

# Receive acknowledgment
PUBACK
```

### Device Receiving RPC

```
# Subscribe to RPC requests
SUBSCRIBE
  Topic: v1/devices/me/rpc/request/+
  QoS: 1

# Receive SUBACK

# Later, receive RPC request
PUBLISH (from server)
  Topic: v1/devices/me/rpc/request/123
  Payload: {"method": "getValue", "params": {}}

# Send response
PUBLISH
  Topic: v1/devices/me/rpc/response/123
  Payload: {"value": 42}
```

## See Also

- [Transport Contract](./transport-contract.md) - Common transport interface
- [Device Entity](../02-core-concepts/entities/device.md) - Device authentication
- [Telemetry](../02-core-concepts/data-model/telemetry.md) - Data format
- [Attributes](../02-core-concepts/data-model/attributes.md) - Attribute handling
- [CoAP Protocol](./coap.md) - Alternative constrained protocol
- [HTTP Protocol](./http.md) - REST-based alternative
