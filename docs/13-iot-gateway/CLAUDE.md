# CLAUDE.md - IoT Gateway Section

This file provides guidance for working with the IoT Gateway documentation section.

## Section Purpose

This section documents the ThingsBoard IoT Gateway:

- **Gateway Architecture**: Python-based bridge for legacy systems
- **Connectors**: Protocol-specific modules (Modbus, OPC-UA, BLE, etc.)
- **Converters**: Data transformation between protocols and ThingsBoard

## File Structure

```
13-iot-gateway/
├── README.md                  # Gateway overview and architecture
├── gateway-architecture.md    # Components, data flow, deployment
└── connectors-overview.md     # Connector types and selection guide
```

## Writing Guidelines

### Audience

Integration engineers connecting legacy and industrial systems. Assume familiarity with industrial protocols but not necessarily with ThingsBoard-specific patterns.

### Content Pattern

Gateway documents should include:

1. **Overview** - What the component does
2. **Architecture** - How it fits in the system
3. **Configuration** - JSON/YAML settings
4. **Data Flow** - Message transformation
5. **Deployment** - Installation options
6. **Pitfalls** - Common integration issues
7. **See Also** - Related documentation

### Connector Documentation Pattern

For each connector type:

```markdown
## Connector Name

**Protocol**: What protocol it supports

### Configuration

\`\`\`json
{
  "name": "connector-name",
  "type": "connector-type",
  "configuration": "connector.json"
}
\`\`\`

### Data Mapping

| Source | ThingsBoard | Transform |
|--------|-------------|-----------|
| ... | ... | ... |

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Connector" for protocol-specific modules
- Use "Converter" for data transformation logic
- Use "Event Storage" for local buffering
- Use "ThingsBoard Client" for MQTT connection to platform
- Use "Gateway Service" for main orchestration component

### Diagrams

Use Mermaid diagrams to show:

- Gateway architecture (`graph TB`)
- Data flow (`sequenceDiagram`)
- Connector topology (`graph LR`)
- Buffering behavior (`sequenceDiagram`)

### Technology-Agnostic Rule

Focus on gateway behavior, not implementation:

**DO**: "The Modbus connector polls registers at configurable intervals"
**DON'T**: "ModbusConnector extends Connector with pymodbus.ModbusTcpClient"

**DO**: "Converters transform protocol data to ThingsBoard telemetry format"
**DON'T**: "CustomConverter implements AbstractConverter.convert() returning dict"

**DO**: "Event storage buffers data during network outages for reliable delivery"
**DON'T**: "SQLite3Storage uses sqlite3.Connection with WAL mode"

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/iot-gateway/` - Official Gateway documentation
- ThingsBoard IoT Gateway GitHub repository (Python source)

## Related Sections

- `02-core-concepts/entities/device.md` - Device data model
- `05-transport-layer/mqtt.md` - MQTT transport
- `12-edge/` - Edge deployment
- `14-integrations/` - Platform integrations

## Common Tasks

### Documenting Connector Configuration

1. Show JSON configuration structure
2. Document all configuration options
3. Include example configurations
4. Document polling intervals and timeouts

### Documenting Converters

1. Explain uplink (device→TB) conversion
2. Explain downlink (TB→device) conversion
3. Show transformation examples
4. Document custom converter development

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/13-iot-gateway/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **iot-engineer** | `/iot-engineer` | IoT architecture, device connectivity |
| **mqtt-expert** | `/mqtt-expert` | MQTT protocol, gateway-platform communication |
| **technical-writer** | `/technical-writer` | Clear connector documentation |

### When to Use Each Skill

- **Documenting connectors**: Use `/iot-engineer` for industrial protocols
- **Explaining MQTT communication**: Use `/mqtt-expert` for protocol details
- **Writing configuration guides**: Use `/technical-writer` for clarity

## Key Gateway Concepts

When documenting gateway, emphasize:

| Concept | Key Points |
|---------|------------|
| **Connectors** | Protocol-specific modules for external systems |
| **Converters** | Transform data between protocol and ThingsBoard formats |
| **Event Storage** | Local buffering for reliable delivery |
| **ThingsBoard Client** | MQTT client for platform communication |
| **Custom Extensions** | User-defined connectors and converters |

## Supported Connectors

| Connector | Protocol | Use Case |
|-----------|----------|----------|
| **Modbus** | Modbus TCP/RTU | Industrial sensors, PLCs, meters |
| **OPC-UA** | OPC Unified Architecture | Industrial automation, SCADA |
| **MQTT** | MQTT 3.1.1/5.0 | External MQTT brokers |
| **BLE** | Bluetooth Low Energy | Wireless sensors, beacons |
| **BACnet** | BACnet/IP | Building automation |
| **CAN** | CAN bus | Automotive, industrial |
| **Request** | HTTP/HTTPS | REST API endpoints |
| **SNMP** | SNMP v1/v2c/v3 | Network equipment |
| **ODBC** | ODBC databases | Database polling |
| **Custom** | Any | User-defined connectors |

## Common Pitfalls to Document

Ensure documentation covers these gateway issues:

| Pitfall | Description |
|---------|-------------|
| Connection timeout | Wrong IP/port or firewall blocking |
| Register mapping | Incorrect Modbus register addresses |
| Data type mismatch | Wrong byte order or data type |
| Polling overload | Too frequent polling overwhelming device |
| Buffer overflow | Event storage growing during long outage |
| Converter errors | Transformation logic exceptions |

## Hardware Documentation

For hardware docs, ensure coverage of:

| Deployment | RAM | CPU | Storage |
|------------|-----|-----|---------|
| Minimum | 256 MB | 1 core | 1 GB |
| Recommended | 512 MB | 2 cores | 4 GB |
| Heavy load | 1+ GB | 4 cores | 10+ GB |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
