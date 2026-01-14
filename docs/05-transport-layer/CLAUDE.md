# CLAUDE.md - Transport Layer Section

This file provides guidance for working with the Transport Layer documentation section.

## Section Purpose

This section documents ThingsBoard's device connectivity layer:

- **Protocol Support**: MQTT, HTTP, CoAP, LwM2M, SNMP
- **Transport Contract**: Common abstraction for all protocols
- **Gateway API**: Multi-device connectivity via MQTT gateway
- **Security**: SSL/TLS, DTLS, certificate management

## File Structure

```
05-transport-layer/
├── README.md              # Transport layer overview and architecture
├── transport-contract.md  # Common abstraction, TransportProtos
├── mqtt.md                # MQTT 3.1.1/5.0 protocol support
├── gateway-mqtt.md        # Multi-device gateway API
├── http.md                # REST-based device API
├── coap.md                # Constrained Application Protocol
├── lwm2m.md               # Lightweight M2M for device management
├── snmp.md                # Simple Network Management Protocol
└── ssl-configuration.md   # TLS/DTLS certificate setup
```

## Writing Guidelines

### Audience

Device developers and IoT integrators connecting devices to ThingsBoard. Assume familiarity with IoT concepts but not necessarily with specific protocol details.

### Content Pattern

Transport documents should include:

1. **Overview** - What the protocol is and when to use it
2. **Connection** - How to establish and maintain connection
3. **Authentication** - Credential types and configuration
4. **Topics/Endpoints** - API structure for data exchange
5. **Message Format** - Payload structure and examples
6. **QoS/Reliability** - Delivery guarantees
7. **Pitfalls** - Common mistakes and solutions
8. **See Also** - Related documentation

### Protocol Documentation Pattern

For each transport protocol:

```markdown
## Protocol Name

**Use Case**: When to choose this protocol

### Connection

| Parameter | Description | Example |
|-----------|-------------|---------|
| Host | ... | ... |
| Port | ... | ... |

### Authentication

| Method | Description |
|--------|-------------|
| Access Token | ... |
| X.509 Certificate | ... |

### Topics/Endpoints

| Topic/Endpoint | Direction | Purpose |
|----------------|-----------|---------|
| ... | Publish/Subscribe | ... |

### Message Format

\`\`\`json
{
  "example": "payload"
}
\`\`\`

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Transport" not "connector" or "adapter"
- Use "Device" not "client" when referring to IoT devices
- Use "Access Token" not "password" for token-based auth
- Use "Topic" for MQTT, "Endpoint" for HTTP/CoAP
- Use "Telemetry" for time-series data, "Attributes" for properties

### Diagrams

Use Mermaid diagrams to show:

- Connection flow (`sequenceDiagram`)
- Protocol architecture (`graph TB`)
- Message routing (`flowchart`)
- Authentication flow (`sequenceDiagram`)

### Technology-Agnostic Rule

Focus on protocol behavior, not implementation:

**DO**: "MQTT transport supports QoS 0, 1, and 2 for message delivery guarantees"
**DON'T**: "MqttTransportHandler extends ChannelInboundHandlerAdapter with Netty pipeline"

**DO**: "Devices authenticate using access tokens passed in the MQTT username field"
**DON'T**: "MqttTransportContext.getCredentials() validates via DeviceCredentialsService"

**DO**: "CoAP uses DTLS for transport-layer security on constrained devices"
**DON'T**: "CoapTransportService configures Californium with DtlsConnectorConfig"

## Protocol Comparison

When documenting, help readers choose the right protocol:

| Protocol | Best For | Constraints |
|----------|----------|-------------|
| **MQTT** | Persistent connections, bidirectional | TCP, moderate overhead |
| **HTTP** | Request/response, firewalls | Higher overhead, no push |
| **CoAP** | Constrained devices, UDP | Limited payload, UDP |
| **LwM2M** | Device management, OTA | Complex setup |
| **SNMP** | Network equipment | Polling-based |

## Reference Sources

When updating this section, cross-reference:

- `ref/thingsboard.github.io-master/docs/reference/` - Official protocol docs
- `ref/thingsboard-master/common/transport/` - Transport abstractions
- `ref/thingsboard-master/transport/mqtt/` - MQTT implementation
- `ref/thingsboard-master/transport/http/` - HTTP implementation
- `ref/thingsboard-master/transport/coap/` - CoAP implementation
- `ref/thingsboard-master/transport/lwm2m/` - LwM2M implementation
- `ref/thingsboard-master/transport/snmp/` - SNMP implementation

## Related Sections

- `02-core-concepts/entities/device.md` - Device provisioning and credentials
- `02-core-concepts/device-profiles.md` - Transport configuration in profiles
- `08-message-queue/` - Transport to core communication
- `09-security/` - Transport authentication and rate limiting
- `13-iot-gateway/` - Gateway architecture for protocol bridging

## Common Tasks

### Documenting a New Transport Feature

1. Identify which protocol file to update
2. Add configuration parameters with examples
3. Include working curl/mosquitto commands
4. Document any version-specific features
5. Add pitfalls section if needed

### Updating Authentication Documentation

1. Verify against source in `ref/thingsboard-master/`
2. Test authentication flow with real credentials
3. Document all supported credential types
4. Include troubleshooting for common auth errors

### Adding Protocol Examples

1. Use real, working examples (test them)
2. Show both minimal and complete payloads
3. Include response examples where applicable
4. Use placeholder tokens like `$ACCESS_TOKEN`

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/05-transport-layer/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **mqtt-expert** | `/mqtt-expert` | MQTT protocol, QoS, topics, session management |
| **grpc-expert** | `/grpc-expert` | TransportProtos, protocol buffers, gRPC |
| **iot-engineer** | `/iot-engineer` | IoT protocols, device connectivity patterns |
| **network-engineer** | `/network-engineer` | Network protocols, TLS/DTLS, firewall configuration |
| **technical-writer** | `/technical-writer` | Clear protocol documentation |

### When to Use Each Skill

- **Documenting MQTT features**: Use `/mqtt-expert` for QoS, topics, sessions
- **Explaining TransportProtos**: Use `/grpc-expert` for protobuf message structure
- **Comparing protocols**: Use `/iot-engineer` for IoT protocol selection
- **Documenting SSL/TLS**: Use `/network-engineer` for certificate and security configuration
- **Writing authentication guides**: Use `/technical-writer` for step-by-step clarity

## Key Transport Concepts

When documenting transports, emphasize:

| Concept | Key Points |
|---------|------------|
| **TransportProtos** | Common message format across all protocols |
| **Session Management** | Connection state, keep-alive, reconnection |
| **Authentication** | Token, X.509, or protocol-specific credentials |
| **Rate Limiting** | Per-device and per-tenant throttling |
| **Message Routing** | Transport → Queue → Rule Engine flow |
| **QoS Mapping** | Protocol QoS to platform guarantees |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Wrong port | Using HTTP port for MQTT or vice versa |
| Token in wrong field | MQTT username vs password confusion |
| Missing Content-Type | HTTP requests without proper headers |
| QoS mismatch | Expecting QoS 2 guarantees from QoS 0 |
| Certificate chain | Incomplete certificate chain for TLS |
| Keep-alive timeout | Connection drops due to idle timeout |
| Payload size limits | Exceeding max message size |

## Security Documentation

For SSL/TLS documentation, cover:

| Aspect | Content |
|--------|---------|
| Certificate types | Self-signed, CA-signed, Let's Encrypt |
| Trust store setup | How to configure trusted CAs |
| Client certificates | X.509 mutual TLS authentication |
| DTLS for UDP | CoAP and LwM2M security |
| Cipher suites | Recommended configurations |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
