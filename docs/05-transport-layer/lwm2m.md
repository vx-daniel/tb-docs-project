# LwM2M Protocol

## Overview

LwM2M (Lightweight Machine-to-Machine) is an OMA (Open Mobile Alliance) standard protocol designed for IoT device management. Built on CoAP, it provides a standardized object model for device registration, configuration, monitoring, and firmware updates. ThingsBoard implements a full LwM2M server supporting versions 1.0-1.2 with comprehensive security modes and OTA capabilities.

## Key Behaviors

1. **Object-Based Model**: Standardized objects for device info, connectivity, firmware, and sensors.

2. **Registration Lifecycle**: Devices register, update registration, and deregister with the server.

3. **Observe Pattern**: Server subscribes to resource changes for real-time monitoring.

4. **Bootstrap Support**: Server-initiated or client-initiated credential provisioning.

5. **Multiple Security Modes**: PSK, RPK, X.509 certificates, and no-security options.

6. **OTA Updates**: Standardized firmware and software update mechanisms.

## Protocol Architecture

### LwM2M Architecture

```mermaid
graph TB
    subgraph "LwM2M Client (Device)"
        OBJ[Object Instances]
        RES[Resources]
        CLIENT[LwM2M Client]
    end

    subgraph "ThingsBoard LwM2M Server"
        SERVER[LwM2M Server]
        BOOTSTRAP[Bootstrap Server]
        REG[Registration Handler]
        OBS[Observation Manager]
    end

    subgraph "ThingsBoard Core"
        TRANSPORT[Transport Service]
        DEVICE[Device Service]
    end

    CLIENT --> |"CoAP/DTLS"| SERVER
    CLIENT --> |"Bootstrap"| BOOTSTRAP
    SERVER --> REG
    SERVER --> OBS
    REG --> TRANSPORT
    OBS --> TRANSPORT
    TRANSPORT --> DEVICE
```

### Object Model Hierarchy

```mermaid
graph TB
    subgraph "LwM2M Path Structure"
        ROOT[/]
        OBJ[/{ObjectID}]
        INST[/{ObjectID}/{InstanceID}]
        RES[/{ObjectID}/{InstanceID}/{ResourceID}]
    end

    ROOT --> OBJ
    OBJ --> INST
    INST --> RES
```

**Path Format:** `/{ObjectID}/{InstanceID}/{ResourceID}`

Example: `/3/0/0` = Device Object, Instance 0, Manufacturer resource

## Standard Objects

### Core Objects

```mermaid
graph TB
    subgraph "Mandatory Objects"
        OBJ0[Object 0<br/>LwM2M Security]
        OBJ1[Object 1<br/>LwM2M Server]
        OBJ3[Object 3<br/>Device]
    end

    subgraph "Optional Objects"
        OBJ2[Object 2<br/>Access Control]
        OBJ4[Object 4<br/>Connectivity]
        OBJ5[Object 5<br/>Firmware Update]
        OBJ9[Object 9<br/>Software Mgmt]
        OBJ19[Object 19<br/>Binary Data]
    end
```

### Object 0: LwM2M Security

| Resource | ID | Description |
|----------|------|-------------|
| Server URI | 0 | Bootstrap/LwM2M server URI |
| Bootstrap Server | 1 | Is bootstrap server flag |
| Security Mode | 2 | PSK (0), RPK (1), Certificate (2), NoSec (3) |
| Public Key | 3 | Client public key or identity |
| Server Public Key | 4 | Server public key |
| Secret Key | 5 | PSK or private key |
| Short Server ID | 10 | Server identifier |

### Object 1: LwM2M Server

| Resource | ID | Description |
|----------|------|-------------|
| Short Server ID | 0 | Server identifier |
| Lifetime | 1 | Registration lifetime (seconds) |
| Default Min Period | 2 | Minimum observation period |
| Default Max Period | 3 | Maximum observation period |
| Disable Timeout | 5 | Disable timeout period |
| Binding | 7 | U (UDP), T (TCP), S (SMS) |

### Object 3: Device

| Resource | ID | Description |
|----------|------|-------------|
| Manufacturer | 0 | Device manufacturer |
| Model Number | 1 | Device model |
| Serial Number | 2 | Serial number |
| Firmware Version | 3 | Current firmware |
| Reboot | 4 | Execute to reboot |
| Factory Reset | 5 | Execute to reset |
| Battery Level | 9 | Battery percentage |
| Memory Total | 21 | Total memory |

### Object 5: Firmware Update

| Resource | ID | Description |
|----------|------|-------------|
| Package | 0 | Firmware binary |
| Package URI | 1 | Download URL |
| Update | 2 | Execute to install |
| State | 3 | Current state |
| Update Result | 5 | Last update result |

## Operations

### Supported Operations

```mermaid
graph TB
    subgraph "Read Operations"
        READ[Read]
        READ_COMP[Read Composite]
        DISCOVER[Discover]
    end

    subgraph "Write Operations"
        WRITE_REP[Write Replace]
        WRITE_UPD[Write Update]
        WRITE_COMP[Write Composite]
        WRITE_ATTR[Write Attributes]
    end

    subgraph "Observe Operations"
        OBSERVE[Observe]
        OBS_COMP[Observe Composite]
        CANCEL[Cancel Observe]
    end

    subgraph "Management"
        EXECUTE[Execute]
        CREATE[Create]
        DELETE[Delete]
    end
```

| Operation | Description | LwM2M Version |
|-----------|-------------|---------------|
| READ | Read single resource value | 1.0+ |
| READ_COMPOSITE | Read multiple resources | 1.1+ |
| DISCOVER | Get resource metadata | 1.0+ |
| OBSERVE | Subscribe to changes | 1.0+ |
| OBSERVE_COMPOSITE | Subscribe to multiple resources | 1.1+ |
| EXECUTE | Execute resource function | 1.0+ |
| WRITE_REPLACE | Replace entire resource | 1.0+ |
| WRITE_UPDATE | Partial update | 1.0+ |
| WRITE_COMPOSITE | Update multiple resources | 1.1+ |
| CREATE | Create object instance | 1.0+ |
| DELETE | Delete object instance | 1.0+ |

## LwM2M Version Support

### Version Differences

| Feature | LwM2M 1.0 | LwM2M 1.1 |
|---------|-----------|-----------|
| Content Format | TLV | TEXT (JSON-like) |
| Composite Operations | Not supported | Supported |
| Object Versioning | Basic | Full support |
| Default Object Version | 1.0 | 1.1 |

**Version Detection:**
- Protocol version from client registration
- Per-client version tracking
- Operation type enforcement based on version

### Object Versioning

Objects can have multiple versions within a deployment:

```
/{ObjectID}.{Version}/{InstanceID}/{ResourceID}
```

Example: `/3.1.0/0/0` = Device Object v1.0, Instance 0, Manufacturer

**Version Handling:**
- Devices declare supported versions in registration links (`ver=1.0`)
- Platform tracks `supportedClientObjects` map
- Resource model validation based on declared version
- Default fallback to version "1.0"

### Composite Operations (LwM2M 1.1)

**Benefits:**
- Multiple resources in single CoAP message
- Reduced message overhead for bulk operations
- More efficient for periodic telemetry collection

**Request Types:**
```mermaid
graph TB
    subgraph "Composite Requests"
        RC[ReadComposite<br/>/3/0/0, /3/0/1, /6/0/2]
        WC[WriteComposite<br/>Multiple paths]
        OC[ObserveComposite<br/>Multiple paths]
        CC[CancelObserveComposite]
    end
```

**Usage:**
```json
{
  "paths": ["/3/0/0", "/3/0/1", "/3/0/9"],
  "operation": "READ_COMPOSITE"
}
```

### Operation Flow

```mermaid
sequenceDiagram
    participant Server as LwM2M Server
    participant Client as LwM2M Client

    Server->>Client: READ /3/0/0
    Client-->>Server: "Device Manufacturer"

    Server->>Client: OBSERVE /3/0/9
    Client-->>Server: 2.05 Content (Battery: 85)

    loop Battery Changes
        Client-->>Server: Notify (Battery: 80)
    end

    Server->>Client: EXECUTE /3/0/4
    Note over Client: Device reboots
    Client-->>Server: 2.04 Changed
```

## Security Modes

### Security Options

```mermaid
graph TB
    subgraph "Security Modes"
        PSK[PSK<br/>Pre-Shared Key]
        RPK[RPK<br/>Raw Public Key]
        X509[X.509<br/>Certificate]
        NOSEC[NO_SEC<br/>No Security]
    end
```

### PSK (Pre-Shared Key)

| Parameter | Description |
|-----------|-------------|
| Identity | PSK identity string |
| Key | Hex-encoded shared secret |

**Use Case:** Simple devices with pre-provisioned keys.

### RPK (Raw Public Key)

| Parameter | Description |
|-----------|-------------|
| Client Public Key | Base64-encoded public key |
| Client Private Key | Base64-encoded private key |
| Server Public Key | Base64-encoded server key |

**Use Case:** Certificate-less authentication for constrained devices.

### X.509 Certificate

| Parameter | Description |
|-----------|-------------|
| Client Certificate | PEM-encoded certificate |
| Client Private Key | Base64-encoded private key |
| Server Certificate | PEM-encoded server certificate |

**Use Case:** Enterprise deployments with PKI infrastructure.

### Security Configuration

```mermaid
sequenceDiagram
    participant Device
    participant Bootstrap as Bootstrap Server
    participant Server as LwM2M Server

    Device->>Bootstrap: Bootstrap Request (DTLS)
    Bootstrap->>Device: Write /0/0 (Security Object)
    Bootstrap->>Device: Write /1/0 (Server Object)
    Bootstrap->>Device: Bootstrap Finish
    Device->>Server: Register (with provisioned credentials)
```

## Bootstrap

### Bootstrap Process

```mermaid
stateDiagram-v2
    [*] --> BootstrapRequest
    BootstrapRequest --> BootstrapInProgress
    BootstrapInProgress --> SecurityProvisioned: Write Security
    SecurityProvisioned --> ServerProvisioned: Write Server
    ServerProvisioned --> BootstrapComplete: Bootstrap Finish
    BootstrapComplete --> Registration
    Registration --> [*]
```

### Bootstrap Modes

| Mode | Description |
|------|-------------|
| Client-Initiated | Device requests bootstrap |
| Server-Initiated | Server triggers bootstrap |
| Bootstrap on Failure | Auto-bootstrap if registration fails |

### Bootstrap Configuration

| Setting | Description |
|---------|-------------|
| Bootstrap Server URI | coaps://host:5686 |
| Bootstrap Security Mode | PSK, RPK, X.509, NO_SEC |
| Client Hold Off Time | Delay before bootstrap |
| Bootstrap Account Timeout | Session timeout |

## Device Profiles

### LwM2M Profile Configuration

```mermaid
graph TB
    subgraph "LwM2M Device Profile"
        CRED[Client Credentials]
        BOOT[Bootstrap Configuration]
        TELEM[Telemetry Mapping]
        OBS_STRAT[Observe Strategy]
        OTA[OTA Configuration]
    end

    CRED --> SECURITY[Security Mode]
    BOOT --> BOOT_SERVER[Bootstrap Server]
    BOOT --> LWM2M_SERVER[LwM2M Server]
    TELEM --> MAPPINGS[Resource Mappings]
    OTA --> FW_STRAT[Firmware Strategy]
```

### Telemetry Mapping

| Setting | Description |
|---------|-------------|
| Key Name | ThingsBoard attribute/telemetry key |
| Resource Path | LwM2M resource path (e.g., /3/0/9) |
| Attribute Flag | Map as attribute |
| Telemetry Flag | Map as telemetry |

### Observe Strategies

| Strategy | Description |
|----------|-------------|
| SINGLE | One resource per observe request |
| COMPOSITE_ALL | All resources in one request |
| COMPOSITE_BY_OBJECT | Grouped by object ID |

## Firmware Updates

### Update Strategies

```mermaid
graph TB
    subgraph "Firmware Strategies"
        OBJ5_BIN[Object 5 Binary<br/>Direct write to /5/0/0]
        OBJ5_URI[Object 5 URI<br/>Write URL to /5/0/1]
        OBJ19[Object 19 Binary<br/>Binary data container]
    end
```

| Strategy | Description | Use Case |
|----------|-------------|----------|
| OBJ_5_BINARY | Write binary to Package resource | Small firmware |
| OBJ_5_TEMP_URL | Write download URL | Large firmware |
| OBJ_19_BINARY | Use Binary Data Container | Complex packages |

### Firmware State Machine

```mermaid
stateDiagram-v2
    Idle --> Downloading: Package URI written
    Downloading --> Downloaded: Download complete
    Downloaded --> Updating: Execute Update
    Updating --> Idle: Update success/failure
```

| State | Value | Description |
|-------|-------|-------------|
| Idle | 0 | Ready for update |
| Downloading | 1 | Download in progress |
| Downloaded | 2 | Ready to install |
| Updating | 3 | Installation in progress |

### Update Results

| Code | Result |
|------|--------|
| 0 | Initial state |
| 1 | Success |
| 2 | Insufficient storage |
| 3 | Out of memory |
| 4 | Connection lost |
| 5 | Integrity check failure |
| 6 | Unsupported package type |
| 7 | Invalid URI |
| 8 | Update failed |
| 9 | Unsupported protocol |
| 10 | Cancelled |

### OTA Resource Paths

| Resource | Firmware (Object 5) | Software (Object 9) |
|----------|---------------------|---------------------|
| Package | /5/0/0 | /9/0/2 |
| Package URI | /5/0/1 | /9/0/3 |
| Execute | /5/0/2 | /9/0/4 |
| State | /5/0/3 | /9/0/7 |
| Result | /5/0/5 | /9/0/9 |

**CoAP Delivery Resources:**
- Firmware: `/tbfw`
- Software: `/tbsw`

### OTA State Tracking

Per-client OTA state is maintained in `LwM2MClientFwOtaInfo` / `LwM2MClientSwOtaInfo`:

| Property | Description |
|----------|-------------|
| currentTitle | Package title |
| currentVersion | Package version |
| targetTitle | Target update title |
| targetVersion | Target update version |
| checksum | SHA-256 verification |
| deliveryMethod | CoAP (1) or CoAPS (2) |
| updateState | Current state enum |
| updateResult | Last result code |

**Device Attribute Mapping:**
- `fw_title`, `fw_version`, `fw_tag`, `fw_url`
- `sw_title`, `sw_version`, `sw_tag`, `sw_url`

## Configuration

### Server Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| bind_address | 0.0.0.0 | Server bind address |
| bind_port | 5685 | LwM2M server port |
| bootstrap_port | 5686 | Bootstrap server port |
| security_mode | DTLS | Default security |
| lifetime | 300 | Default registration lifetime |

### DTLS Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| retransmission_timeout | 9000ms | DTLS retransmit (RFC7925) |
| connection_id_length | 8 | RFC 9146 Connection ID |
| max_transmission_unit | 1024 | Max DTLS fragment |
| max_fragment_length | 1024 | Max CoAP message |

**Supported Cipher Suites:**

| Security Mode | Cipher Suite |
|---------------|--------------|
| PSK | TLS_PSK_WITH_AES_128_CCM_8 |
| PSK | TLS_PSK_WITH_AES_128_CBC_SHA256 |
| RPK/X.509 | TLS_ECDHE_ECDSA_* |

**Authentication Protocols:**

| Protocol | OID |
|----------|-----|
| MD5 | 1.3.6.1.6.3.10.1.1.2 |
| SHA-1 | 1.3.6.1.6.3.10.1.1.3 |
| SHA-224 | 1.3.6.1.6.3.10.1.1.4 |
| SHA-256 | 1.3.6.1.6.3.10.1.1.5 |
| SHA-384 | 1.3.6.1.6.3.10.1.1.6 |
| SHA-512 | 1.3.6.1.6.3.10.1.1.7 |

**Privacy Protocols:**

| Protocol | OID |
|----------|-----|
| DES | 1.3.6.1.6.3.10.1.2.2 |
| AES-128 | 1.3.6.1.6.3.10.1.2.4 |
| AES-192 | 1.3.6.1.4.1.4976.2.2.1.1.1 |
| AES-256 | 1.3.6.1.4.1.4976.2.2.1.1.2 |

### Storage Configuration

| Store | Default | Description |
|-------|---------|-------------|
| Registration Store | In-Memory | Client registration data |
| Security Store | In-Memory | PSK/RPK/X.509 credentials |
| DTLS Session Store | In-Memory | DTLS session state |
| Model Config Store | In-Memory | LwM2M model configurations |

**Redis-Backed Storage:**
For clustered deployments, stores can be backed by Redis:
- `TbLwM2mRedisSecurityStore`
- `TbLwM2mRedisSessionStore`
- `TbLwM2mRedisModelConfigStore`

### Thread Pool Configuration

| Pool | Default | Purpose |
|------|---------|---------|
| Downlink Executor | 8 | Server-to-device operations |
| Uplink Executor | 8 | Device-to-server processing |
| OTA Executor | 4 | Firmware/software updates |

## Registration Lifecycle

### Registration Flow

```mermaid
sequenceDiagram
    participant Client as LwM2M Client
    participant Server as LwM2M Server
    participant TB as ThingsBoard

    Client->>Server: Register (endpoint, lifetime, objects)
    Server->>TB: Create/Update device
    Server-->>Client: 2.01 Created

    loop Registration Update
        Client->>Server: Update (before lifetime expires)
        Server-->>Client: 2.04 Changed
    end

    Client->>Server: Deregister
    Server->>TB: Update device status
    Server-->>Client: 2.02 Deleted
```

### Registration Parameters

| Parameter | Description |
|-----------|-------------|
| Endpoint Name | Unique client identifier |
| Lifetime | Registration validity (seconds) |
| LwM2M Version | Protocol version |
| Binding Mode | U (UDP), T (TCP) |
| Objects | Supported objects list |

## Best Practices

### Device Implementation

| Practice | Benefit |
|----------|---------|
| Use Observe | Efficient updates |
| Implement Object 3 | Device info standardized |
| Support firmware update | Enable OTA |
| Use DTLS | Secure communication |

### Server Configuration

| Practice | Benefit |
|----------|---------|
| Configure timeouts | Handle unreliable networks |
| Map resources to telemetry | Automatic data collection |
| Use composite observe | Reduce traffic |
| Enable bootstrap | Simplified provisioning |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Registration fails | Wrong credentials | Verify security config |
| Observe not working | Client doesn't support | Check object support |
| Firmware update fails | Wrong strategy | Match device capability |
| DTLS handshake fails | Certificate issue | Verify certificate chain |

### Debugging

| Tool | Purpose |
|------|---------|
| Wireshark | CoAP/DTLS packet analysis |
| Debug logging | Server-side troubleshooting |
| Device logs | Client-side issues |

## See Also

- [Transport Contract](./transport-contract.md) - Common transport behaviors
- [CoAP Protocol](./coap.md) - Underlying protocol
- [Device Entity](../02-core-concepts/entities/device.md) - Device configuration
- [Authentication](../06-api-layer/authentication.md) - Security overview
