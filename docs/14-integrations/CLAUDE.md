# CLAUDE.md - Platform Integrations Section

This file provides guidance for working with the Platform Integrations documentation section.

## Section Purpose

This section documents ThingsBoard Platform Integrations:

- **Cloud Integrations**: AWS IoT, Azure IoT Hub, Google Pub/Sub
- **LoRaWAN Integrations**: ChirpStack, TTN, Loriot, ThingPark, Sigfox
- **Messaging Integrations**: HTTP, Kafka, RabbitMQ, MQTT brokers

## File Structure

```
14-integrations/
├── README.md                   # Integration overview and architecture
├── cloud-integrations.md       # AWS, Azure, Google Cloud integrations
├── lorawan-integrations.md     # LoRaWAN network server integrations
└── messaging-integrations.md   # HTTP, Kafka, RabbitMQ, MQTT integrations
```

## Writing Guidelines

### Audience

Integration engineers connecting external IoT platforms and network servers. Assume familiarity with cloud IoT platforms but not necessarily with ThingsBoard-specific patterns.

### Content Pattern

Integration documents should include:

1. **Overview** - What the integration does
2. **Configuration** - Setup parameters
3. **Converters** - Uplink and downlink transformation
4. **Data Flow** - Message routing
5. **Deployment** - Embedded vs remote options
6. **Pitfalls** - Common integration issues
7. **See Also** - Related documentation

### Integration Documentation Pattern

For each integration type:

```markdown
## Integration Name

**Protocol**: Connection protocol

### Configuration

| Parameter | Description | Required |
|-----------|-------------|----------|
| ... | ... | ... |

### Uplink Converter

```javascript
// Example decoder
var data = decodeToJson(payload);
return {
    deviceName: data.deviceId,
    telemetry: { temperature: data.temp }
};
```

### Downlink Converter

```javascript
// Example encoder
return {
    contentType: "JSON",
    data: JSON.stringify(msg)
};
```

### Common Pitfalls

| Pitfall | Impact | Solution |
|---------|--------|----------|
| ... | ... | ... |
```

### Terminology

- Use "Uplink Converter" for device→ThingsBoard transformation
- Use "Downlink Converter" for ThingsBoard→device transformation
- Use "Integration" for the connection component
- Use "Typed Converter" for UI-configured LoRaWAN converters
- Use "Remote Integration" for on-premises deployment

### Diagrams

Use Mermaid diagrams to show:

- Integration architecture (`graph TB`)
- Data flow (`sequenceDiagram`)
- Converter processing (`sequenceDiagram`)
- Deployment options (`graph LR`)

### Technology-Agnostic Rule

Focus on integration behavior, not implementation:

**DO**: "The uplink converter transforms incoming messages to ThingsBoard telemetry format"
**DON'T**: "AbstractUpLinkConverter.convert() calls decodeToJson() via JsonUtils"

**DO**: "Messages are routed to the rule engine after successful conversion"
**DON'T**: "IntegrationController calls TbMsgCallback.onSuccess() to post to tb_rule_engine topic"

**DO**: "Typed converters simplify configuration for supported LoRaWAN networks"
**DON'T**: "ChirpStackIntegration extends AbstractLoRaWanIntegration with TypedUplinkConverter"

## Reference Sources

When updating this section, cross-reference:

- `ref/thingsboard.github.io-master/docs/user-guide/integrations/` - Official Integration docs
- `ref/thingsboard-master/application/src/main/java/org/thingsboard/server/service/integration/` - Integration services

## Related Sections

- `04-rule-engine/` - Message processing after integration
- `13-iot-gateway/` - Local device integration
- `05-transport-layer/` - Direct device protocols
- `12-edge/` - Edge deployment

## Common Tasks

### Documenting Uplink Converters

1. Show input payload structure
2. Document required output fields (deviceName, deviceType)
3. Include transformation examples
4. Document error handling

### Documenting Downlink Converters

1. Explain trigger conditions
2. Show output format requirements
3. Document sync vs async behavior
4. Include rule chain configuration

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/14-integrations/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **iot-engineer** | `/iot-engineer` | IoT integration patterns |
| **kafka-expert** | `/kafka-expert` | Kafka integration configuration |
| **mqtt-expert** | `/mqtt-expert` | MQTT broker integration |
| **technical-writer** | `/technical-writer` | Clear integration documentation |

### When to Use Each Skill

- **Documenting cloud integrations**: Use `/iot-engineer` for platform patterns
- **Explaining Kafka integration**: Use `/kafka-expert` for configuration
- **Documenting MQTT integration**: Use `/mqtt-expert` for protocol details
- **Writing converter guides**: Use `/technical-writer` for clarity

## Key Integration Concepts

When documenting integrations, emphasize:

| Concept | Key Points |
|---------|------------|
| **Uplink Converter** | Transform incoming data to ThingsBoard format |
| **Downlink Converter** | Transform outgoing commands to platform format |
| **Typed Converter** | UI-based configuration for supported platforms |
| **Remote Integration** | On-premises deployment for local access |
| **Converters Library** | Built-in decoders for 100+ device types |
| **Debug Mode** | Capture messages for troubleshooting |

## Common Pitfalls to Document

Ensure documentation covers these integration issues:

| Pitfall | Description |
|---------|-------------|
| Converter null checks | Missing null checks causing runtime errors |
| Device name extraction | Incorrect device identifier extraction |
| Debug mode enabled | Database growth from debug event storage |
| Credential mismatch | Wrong API keys or certificates |
| Sync vs async downlink | HTTP/Sigfox require uplink before downlink |
| Nested objects | Nested telemetry objects not stored correctly |

## Integration Types

| Category | Integrations |
|----------|--------------|
| **Cloud IoT** | AWS IoT, Azure IoT Hub, Google Pub/Sub |
| **LoRaWAN** | ChirpStack, TTN, Loriot, ThingPark, Sigfox |
| **Messaging** | HTTP, Kafka, RabbitMQ, MQTT, Pulsar |
| **Cellular** | Sigfox, OceanConnect, T-Mobile IoT |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
