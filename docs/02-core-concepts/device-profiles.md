# Device Profiles

## Overview

Device profiles enable centralized management of common settings for multiple devices. Rather than configuring each device individually, administrators define profiles that specify transport configuration, alarm rules, default rule chains, firmware versions, and provisioning strategies. When devices are assigned to a profile, they inherit these settings automatically, simplifying fleet management at scale.

## Key Behaviors

1. **Centralized Configuration**: Define settings once, apply to many devices.

2. **Transport Configuration**: Protocol-specific settings (MQTT topics, payload formats, CoAP modes).

3. **Alarm Rules**: Profile-level alarm conditions with severity and propagation.

4. **Provisioning Strategy**: Control how new devices register via the profile.

5. **OTA Management**: Assign firmware/software versions to profile for fleet updates.

6. **Queue Assignment**: Route device messages to specific processing queues.

## Profile Architecture

### Profile Hierarchy

```mermaid
graph TB
    subgraph "Device Profile"
        PROFILE[Device Profile]
        TRANSPORT[Transport Config]
        ALARMS[Alarm Rules]
        PROVISION[Provision Settings]
        QUEUE[Queue Assignment]
        CHAIN[Default Rule Chain]
        OTA[OTA Packages]
    end

    subgraph "Devices"
        D1[Device 1]
        D2[Device 2]
        D3[Device N]
    end

    PROFILE --> TRANSPORT
    PROFILE --> ALARMS
    PROFILE --> PROVISION
    PROFILE --> QUEUE
    PROFILE --> CHAIN
    PROFILE --> OTA

    PROFILE --> D1
    PROFILE --> D2
    PROFILE --> D3
```

### Configuration Inheritance

```mermaid
graph TB
    PROFILE[Device Profile] --> |"Inherits"| DEVICE[Device]

    subgraph "Profile Settings"
        P_TRANSPORT[Transport Config]
        P_ALARMS[Alarm Rules]
        P_QUEUE[Queue]
        P_CHAIN[Rule Chain]
        P_FW[Firmware]
    end

    subgraph "Device Overrides"
        D_LABEL[Label]
        D_CREDS[Credentials]
        D_ATTRS[Attributes]
        D_FW[Firmware Override]
    end

    P_TRANSPORT --> DEVICE
    P_ALARMS --> DEVICE
    P_QUEUE --> DEVICE
    P_CHAIN --> DEVICE
    P_FW --> DEVICE

    D_LABEL --> DEVICE
    D_CREDS --> DEVICE
    D_ATTRS --> DEVICE
    D_FW --> |"Optional Override"| DEVICE
```

## Profile Settings

### Basic Configuration

| Setting | Required | Description |
|---------|----------|-------------|
| Name | Yes | Unique profile name |
| Description | No | Profile description |
| Default Rule Chain | No | Rule chain for processing messages |
| Default Queue | No | Message queue (default: "Main") |
| Default Dashboard | No | Dashboard for device details |
| Image | No | Profile icon/image |

### Default Rule Chain

Assign a specific rule chain to process all messages from devices using this profile:

```mermaid
graph LR
    DEVICE[Device] --> |"Messages"| PROFILE_CHAIN[Profile Rule Chain]
    PROFILE_CHAIN --> |"Process"| ACTIONS[Actions]

    subgraph "Without Profile Chain"
        DEFAULT[Default Rule Chain] --> |"Routes by type"| CUSTOM[Custom Chains]
    end
```

**Benefits:**
- Simplifies root rule chain
- Isolates processing logic per device type
- Easier maintenance and debugging

### Queue Assignment

Route messages to specific queues for processing isolation:

| Queue | Use Case |
|-------|----------|
| Main | Default queue for general devices |
| HighPriority | Critical alerts (fire alarms, safety) |
| Batch | High-volume, delay-tolerant data |

```mermaid
graph TB
    subgraph "Device Profiles"
        P1[Fire Alarm Profile] --> Q1[HighPriority Queue]
        P2[Sensor Profile] --> Q2[Main Queue]
        P3[Meter Profile] --> Q3[Batch Queue]
    end

    subgraph "Processing"
        Q1 --> |"Immediate"| RE1[Rule Engine]
        Q2 --> |"Normal"| RE2[Rule Engine]
        Q3 --> |"Delayed"| RE3[Rule Engine]
    end
```

## Transport Configuration

### Transport Types

```mermaid
graph TB
    subgraph "Transport Types"
        DEFAULT[Default<br/>HTTP/MQTT/CoAP]
        MQTT[MQTT<br/>Custom topics, Protobuf]
        COAP[CoAP<br/>Default, Efento]
        LWM2M[LwM2M<br/>Object model]
        SNMP[SNMP<br/>OID mapping]
    end
```

### Default Transport

Basic transport supporting standard HTTP, MQTT, and CoAP APIs with JSON payloads.

| Setting | Description |
|---------|-------------|
| Type | Default |
| Payload | JSON |

### MQTT Transport

Custom MQTT topic filters and payload formats:

| Setting | Description | Example |
|---------|-------------|---------|
| Telemetry Topic Filter | Custom topic for telemetry | `/sensors/+/telemetry` |
| Attributes Topic Filter | Custom topic for attributes | `/sensors/+/attributes` |
| Payload Type | JSON or Protobuf | Protobuf |

**Topic Wildcards:**
- `+` - Single level wildcard
- `#` - Multi-level wildcard

**Example Configuration:**
```
Telemetry Topic: /devices/${deviceName}/data
Attributes Topic: /devices/${deviceName}/attrs
```

### MQTT Payload Formats

| Format | Description | Use Case |
|--------|-------------|----------|
| JSON | Human-readable, larger | Development, debugging |
| Protobuf | Binary, compact | Production, bandwidth-limited |

**Protobuf Schema Definition:**

```protobuf
syntax = "proto3";

message TelemetryData {
  double temperature = 1;
  double humidity = 2;
  int64 timestamp = 3;
}
```

**Compatibility Mode:**
- Tries Protobuf first, falls back to JSON
- Useful during firmware migration
- Slight performance overhead

### CoAP Transport

| Device Type | Description |
|-------------|-------------|
| Default | Standard CoAP with JSON/Protobuf |
| Efento NB-IoT | Specialized for Efento sensors |

**Power Saving Modes:**

| Mode | Description |
|------|-------------|
| PSM | Power Saving Mode - extended sleep |
| DRX | Discontinuous Reception |
| eDRX | Extended DRX |

### LwM2M Transport

Standardized object model configuration:

| Setting | Description |
|---------|-------------|
| Object Model | LwM2M object definitions |
| Observe Strategy | SINGLE, COMPOSITE_ALL, COMPOSITE_BY_OBJECT |
| Bootstrap | Bootstrap server settings |

See [LwM2M Protocol](../05-transport-layer/lwm2m.md) for detailed configuration.

### SNMP Transport

OID mapping and polling configuration:

| Setting | Description |
|---------|-------------|
| Timeout | Request timeout (ms) |
| Retries | Retry attempts |
| Communication Configs | OID mappings and polling frequency |

See [SNMP Protocol](../05-transport-layer/snmp.md) for detailed configuration.

## Alarm Rules

### Alarm Rule Structure

```mermaid
graph TB
    subgraph "Alarm Rule"
        CONDITION[Create Condition]
        CLEAR[Clear Condition]
        SEVERITY[Severity]
        PROPAGATE[Propagation]
        DETAILS[Alarm Details]
    end

    CONDITION --> |"Triggers"| ALARM[Alarm]
    CLEAR --> |"Clears"| ALARM
    ALARM --> SEVERITY
    ALARM --> PROPAGATE
    ALARM --> DETAILS
```

### Alarm Configuration

| Setting | Description |
|---------|-------------|
| Alarm Type | Unique alarm identifier |
| Create Condition | Condition that triggers alarm |
| Clear Condition | Condition that clears alarm |
| Severity | CRITICAL, MAJOR, MINOR, WARNING, INDETERMINATE |
| Propagate | Propagate to related entities |
| Details | Dynamic alarm details function |

### Condition Types

| Type | Description | Example |
|------|-------------|---------|
| Simple | Single key comparison | `temperature > 100` |
| Duration | Condition over time | `temperature > 100 for 5 minutes` |
| Repeating | Multiple occurrences | `temperature > 100, 3 times in 5 minutes` |

### Condition Keys

| Key Source | Description |
|------------|-------------|
| Telemetry | Time-series values |
| Attribute | Device attributes |
| Constant | Fixed values |
| Entity Count | Alarm entity count |

### Condition Operations

| Category | Operations |
|----------|------------|
| Numeric | `>`, `<`, `>=`, `<=`, `==`, `!=` |
| String | `EQUAL`, `NOT_EQUAL`, `STARTS_WITH`, `ENDS_WITH`, `CONTAINS` |
| Boolean | `==`, `!=` |
| Complex | `AND`, `OR` |

### Alarm Details Function

Dynamic alarm details using TBEL:

```javascript
var details = {};
details.temperature = $root.temperature;
details.threshold = 100;
details.message = 'Temperature exceeded threshold: ' + $root.temperature;
return details;
```

### Example: Temperature Alarm

```json
{
  "alarmType": "High Temperature",
  "createRules": {
    "CRITICAL": {
      "condition": {
        "spec": {
          "type": "SIMPLE"
        },
        "condition": [{
          "key": {
            "type": "TIME_SERIES",
            "key": "temperature"
          },
          "predicate": {
            "type": "NUMERIC",
            "operation": "GREATER",
            "value": {
              "defaultValue": 100
            }
          }
        }]
      }
    }
  },
  "clearRule": {
    "condition": {
      "spec": {
        "type": "SIMPLE"
      },
      "condition": [{
        "key": {
          "type": "TIME_SERIES",
          "key": "temperature"
        },
        "predicate": {
          "type": "NUMERIC",
          "operation": "LESS_OR_EQUAL",
          "value": {
            "defaultValue": 100
          }
        }
      }]
    }
  },
  "propagate": true,
  "propagateToOwner": true
}
```

## Device Provisioning

### Provisioning Settings

| Setting | Description |
|---------|-------------|
| Provision Strategy | Allow new / Check pre-provisioned |
| Provision Device Key | Unique profile key |
| Provision Device Secret | Authentication secret |

See [Device Provisioning](./device-provisioning.md) for detailed configuration.

## OTA Updates

### Firmware Assignment

Assign firmware packages to the profile for fleet-wide updates:

```mermaid
graph TB
    FW[Firmware Package] --> PROFILE[Device Profile]
    PROFILE --> D1[Device 1]
    PROFILE --> D2[Device 2]
    PROFILE --> D3[Device N]

    D1 --> |"Notified"| UPDATE1[Update]
    D2 --> |"Notified"| UPDATE2[Update]
    D3 --> |"Notified"| UPDATE3[Update]
```

### Software Assignment

Same as firmware but for software packages (Object 9 in LwM2M).

See [OTA Updates](./ota-updates.md) for detailed configuration.

## Profile Management

### Create Profile

```mermaid
sequenceDiagram
    participant Admin
    participant TB as ThingsBoard

    Admin->>TB: Create Device Profile
    TB->>TB: Validate settings
    TB->>TB: Store profile
    TB-->>Admin: Profile created

    Admin->>TB: Assign devices to profile
    TB->>TB: Apply profile settings
    TB-->>Admin: Devices updated
```

### REST API

**Create Profile:**
```bash
curl -X POST \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Temperature Sensors",
    "description": "Profile for temperature monitoring devices",
    "type": "DEFAULT",
    "transportType": "MQTT",
    "defaultQueueName": "Main"
  }' \
  "https://thingsboard.example.com/api/deviceProfile"
```

**Get Profile:**
```bash
curl -X GET \
  -H "Authorization: Bearer $JWT_TOKEN" \
  "https://thingsboard.example.com/api/deviceProfile/$PROFILE_ID"
```

**List Profiles:**
```bash
curl -X GET \
  -H "Authorization: Bearer $JWT_TOKEN" \
  "https://thingsboard.example.com/api/deviceProfiles?pageSize=10&page=0"
```

### Default Profile

Every tenant has a "default" device profile that cannot be deleted. New devices without an explicit profile assignment use this default.

## Profile Data Model

### Entity Structure

```mermaid
classDiagram
    class DeviceProfile {
        +UUID id
        +UUID tenantId
        +String name
        +String description
        +boolean isDefault
        +DeviceProfileType type
        +DeviceTransportType transportType
        +String defaultQueueName
        +UUID defaultRuleChainId
        +UUID defaultDashboardId
        +UUID firmwareId
        +UUID softwareId
        +DeviceProfileData profileData
    }

    class DeviceProfileData {
        +AlarmSettings alarmSettings
        +DeviceProfileConfiguration configuration
        +DeviceProfileTransportConfiguration transportConfiguration
        +DeviceProfileProvisionConfiguration provisionConfiguration
    }

    DeviceProfile --> DeviceProfileData
```

### Database Schema

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| tenant_id | uuid | Owning tenant |
| name | varchar | Profile name |
| type | varchar | Profile type |
| transport_type | varchar | Transport protocol |
| provision_device_key | varchar | Provisioning key |
| default_rule_chain_id | uuid | FK to rule chain |
| default_queue_name | varchar | Queue name |
| firmware_id | uuid | FK to OTA package |
| software_id | uuid | FK to OTA package |
| profile_data | jsonb | Configuration data |

## Use Cases

### Industrial Sensors

```mermaid
graph TB
    subgraph "Industrial Sensor Profile"
        MODBUS[Transport: Default]
        HIGH_ALARM[Alarm: High Temp > 80°C]
        CRIT_ALARM[Alarm: Critical > 100°C]
        POLL[Queue: HighPriority]
    end
```

**Configuration:**
- Transport: Default (JSON)
- Alarms: Temperature thresholds
- Queue: HighPriority for immediate processing
- Rule Chain: Industrial processing chain

### Smart Meters

```mermaid
graph TB
    subgraph "Smart Meter Profile"
        MQTT[Transport: MQTT Custom]
        BATCH[Queue: Batch]
        DAILY[Alarm: Daily consumption]
    end
```

**Configuration:**
- Transport: MQTT with custom topics
- Queue: Batch for high-volume data
- Alarms: Consumption anomalies
- Firmware: Smart meter firmware

### LwM2M Devices

```mermaid
graph TB
    subgraph "LwM2M Profile"
        LWM2M_TRANS[Transport: LwM2M]
        OBSERVE[Observe: Composite]
        OTA[OTA: Object 5/9]
    end
```

**Configuration:**
- Transport: LwM2M with object model
- Observe Strategy: Composite for efficiency
- Bootstrap: Enabled for provisioning
- Security: PSK or X.509

## Best Practices

### Profile Organization

| Practice | Benefit |
|----------|---------|
| One profile per device type | Clear separation |
| Descriptive names | Easy identification |
| Document alarm rules | Maintainability |
| Test before deployment | Prevent errors |

### Transport Configuration

| Practice | Benefit |
|----------|---------|
| Use Protobuf for production | Bandwidth efficiency |
| Custom topics per device type | Clean topic structure |
| Enable compatibility during migration | Smooth transitions |

### Alarm Configuration

| Practice | Benefit |
|----------|---------|
| Use duration conditions | Reduce false positives |
| Set appropriate severities | Proper prioritization |
| Configure propagation | Related entity visibility |
| Include details function | Rich alarm context |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Device not using profile | Wrong profile assigned | Verify device profile assignment |
| Alarms not triggering | Condition not met | Check telemetry values and conditions |
| Custom topics not working | Topic filter mismatch | Verify wildcard patterns |
| Protobuf parsing fails | Schema mismatch | Validate proto schema |

### Debug Steps

1. Verify device profile assignment
2. Check transport configuration matches device
3. Review alarm rule conditions
4. Test with debug logging enabled
5. Validate message format (JSON/Protobuf)

## See Also

- [Device Entity](./entities/device.md) - Device configuration
- [Device Provisioning](./device-provisioning.md) - Automatic device registration
- [OTA Updates](./ota-updates.md) - Firmware/software distribution
- [Alarm Entity](./entities/alarm.md) - Alarm data model
- [Rule Engine](../04-rule-engine/README.md) - Message processing
- [Transport Layer](../05-transport-layer/README.md) - Protocol details
