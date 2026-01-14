# Messaging Integrations

## Overview

Messaging integrations connect ThingsBoard to external message brokers, streaming platforms, and HTTP endpoints. These integrations enable data ingestion from MQTT brokers, Apache Kafka, RabbitMQ, Apache Pulsar, and HTTP webhooks. They support both consuming messages from external systems and publishing data back, making them ideal for enterprise integration scenarios and stream processing pipelines.

## Supported Platforms

| Platform | Protocol | Direction | Use Case |
|----------|----------|-----------|----------|
| HTTP | HTTP/HTTPS | Bidirectional | Webhooks, REST APIs |
| MQTT | MQTT 3.1.1/5.0 | Bidirectional | External MQTT brokers |
| Apache Kafka | Kafka | Bidirectional | Stream processing |
| RabbitMQ | AMQP | Bidirectional | Message queuing |
| Apache Pulsar | Pulsar | Bidirectional | Distributed messaging |

---

## HTTP Integration

HTTP integration receives data via webhooks and can send HTTP requests to external systems.

```mermaid
graph LR
    subgraph "External Systems"
        DEVICES[Devices/Services]
        API[External API]
    end

    subgraph "ThingsBoard"
        INT[HTTP Integration]
        CONV[Converters]
        TB[ThingsBoard Platform]
    end

    DEVICES -->|POST Webhook| INT
    INT --> CONV
    CONV --> TB
    TB --> CONV
    CONV --> INT
    INT -->|HTTP Request| API
```

### Key Configuration

| Parameter | Description |
|-----------|-------------|
| Base URL | ThingsBoard server URL |
| HTTP Endpoint URL | Generated webhook URL for receiving data |
| Headers filter | Optional security headers for validation |
| Downlink URL | Target URL for outbound requests |

### Use Cases

- Receive webhooks from third-party services
- Connect devices with custom HTTP-based protocols
- Stream data from external IoT platforms
- Bridge systems that only support HTTP callbacks

### Uplink Converter Example

```javascript
var data = decodeToJson(payload);

var result = {
    deviceName: data.deviceId || metadata['deviceName'],
    deviceType: data.deviceType || 'http-device',
    telemetry: {
        temperature: data.temp,
        humidity: data.hum
    },
    attributes: {
        model: data.model
    }
};

return result;
```

### Downlink Configuration

HTTP integration supports synchronous downlinks. The response to an uplink webhook contains any queued downlink data.

```javascript
var result = {
    contentType: "JSON",
    data: JSON.stringify({
        command: msg.method,
        params: msg.params
    }),
    metadata: {}
};

return result;
```

### Security Options

| Option | Description |
|--------|-------------|
| Headers filter | Require specific header values for authentication |
| HTTPS | Enable TLS encryption for webhook endpoint |
| IP whitelist | Restrict incoming connections by IP address |

---

## MQTT Integration

Connects to external MQTT brokers to consume device data and publish commands.

```mermaid
graph LR
    subgraph "External MQTT"
        DEVICES[IoT Devices]
        BROKER[MQTT Broker]
    end

    subgraph "ThingsBoard"
        INT[MQTT Integration]
        CONV[Converters]
        TB[ThingsBoard Platform]
    end

    DEVICES -->|Publish| BROKER
    BROKER -->|Subscribe| INT
    INT --> CONV
    CONV --> TB
    TB --> CONV
    CONV --> INT
    INT -->|Publish| BROKER
```

### Key Configuration

| Parameter | Description |
|-----------|-------------|
| Host | MQTT broker hostname |
| Port | Broker port (1883/8883) |
| Client ID | MQTT client identifier |
| Username/Password | Broker authentication |
| SSL/TLS | Enable secure connection |
| Topic filters | Topics to subscribe |
| QoS | Quality of service level |

### Topic Mapping

Map MQTT topics to device names and data types:

```json
{
  "topicFilter": "sensors/+/telemetry",
  "converter": {
    "type": "custom",
    "deviceNamePattern": "${topic[1]}",
    "deviceTypePattern": "mqtt-sensor"
  }
}
```

**Topic Pattern Examples:**

| Pattern | Matches | Device Name |
|---------|---------|-------------|
| `sensors/+/telemetry` | sensors/device1/telemetry | device1 |
| `building/#` | building/floor1/room2/temp | floor1-room2 |
| `+/+/data` | site/dev123/data | dev123 |

### Uplink Converter Example

```javascript
var data = decodeToJson(payload);
var topicParts = metadata.topic.split('/');

var result = {
    deviceName: topicParts[1],
    deviceType: 'mqtt-sensor',
    telemetry: {
        temperature: data.temp,
        humidity: data.hum
    }
};

return result;
```

### Downlink Configuration

Publish to external broker when rule engine triggers:

```javascript
var result = {
    contentType: "JSON",
    data: JSON.stringify({
        command: msg.method,
        params: msg.params
    }),
    metadata: {
        topic: "devices/" + metadata.deviceName + "/commands"
    }
};

return result;
```

---

## Apache Kafka Integration

Consumes and produces messages to Apache Kafka topics.

```mermaid
graph LR
    subgraph "Kafka Cluster"
        TOPIC_IN[Inbound Topic]
        TOPIC_OUT[Outbound Topic]
    end

    subgraph "ThingsBoard"
        INT[Kafka Integration]
        TB[ThingsBoard Platform]
    end

    TOPIC_IN --> INT
    INT --> TB
    TB --> INT
    INT --> TOPIC_OUT
```

### Key Configuration

| Parameter | Description |
|-----------|-------------|
| Bootstrap servers | Kafka broker addresses |
| Topic | Topic to consume from |
| Group ID | Consumer group identifier |
| Auto offset reset | earliest/latest |
| Key deserializer | Key format |
| Value deserializer | Value format |

### SASL/SSL Configuration

| Parameter | Description |
|-----------|-------------|
| Security protocol | PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL |
| SASL mechanism | PLAIN, SCRAM-SHA-256, SCRAM-SHA-512 |
| SASL username | Authentication username |
| SASL password | Authentication password |
| SSL truststore | Truststore for TLS |

### Consumer Configuration

```json
{
  "bootstrapServers": "kafka1:9092,kafka2:9092",
  "topic": "iot-telemetry",
  "groupId": "thingsboard-integration",
  "autoOffsetReset": "latest",
  "pollInterval": 1000,
  "consumerProperties": {
    "max.poll.records": 500
  }
}
```

### Uplink Converter Example

```javascript
var data = decodeToJson(payload);

var result = {
    deviceName: data.deviceId,
    deviceType: 'kafka-device',
    telemetry: {
        ts: data.timestamp,
        values: data.measurements
    }
};

return result;
```

### Producer (Downlink) Configuration

Send rule engine messages to Kafka:

```javascript
var result = {
    contentType: "JSON",
    data: JSON.stringify({
        deviceId: metadata.deviceName,
        command: msg.method,
        params: msg.params,
        timestamp: Date.now()
    }),
    metadata: {
        key: metadata.deviceName  // Kafka message key
    }
};

return result;
```

---

## RabbitMQ Integration

Integrates with RabbitMQ message broker using AMQP protocol.

```mermaid
graph LR
    subgraph "RabbitMQ"
        EXCHANGE[Exchange]
        QUEUE_IN[Inbound Queue]
        QUEUE_OUT[Outbound Queue]
    end

    subgraph "ThingsBoard"
        INT[RabbitMQ Integration]
        TB[ThingsBoard Platform]
    end

    EXCHANGE --> QUEUE_IN
    QUEUE_IN --> INT
    INT --> TB
    TB --> INT
    INT --> EXCHANGE
    EXCHANGE --> QUEUE_OUT
```

### Key Configuration

| Parameter | Description |
|-----------|-------------|
| Host | RabbitMQ server hostname |
| Port | AMQP port (5672/5671) |
| Virtual host | RabbitMQ virtual host |
| Username | Authentication username |
| Password | Authentication password |
| Queue name | Queue to consume from |
| Exchange name | Exchange for publishing |
| Routing key | Message routing key |

### Queue Binding

```json
{
  "host": "rabbitmq.example.com",
  "port": 5672,
  "virtualHost": "/",
  "username": "thingsboard",
  "password": "secret",
  "queueName": "iot-data",
  "exchangeName": "iot-exchange",
  "routingKey": "telemetry.#"
}
```

### Message Patterns

| Pattern | Use Case |
|---------|----------|
| Direct | Point-to-point messaging |
| Topic | Wildcard routing |
| Fanout | Broadcast to all queues |
| Headers | Attribute-based routing |

---

## Apache Pulsar Integration

Integrates with Apache Pulsar distributed messaging platform.

```mermaid
graph LR
    subgraph "Pulsar Cluster"
        TOPIC_IN[Input Topic]
        TOPIC_OUT[Output Topic]
    end

    subgraph "ThingsBoard"
        INT[Pulsar Integration]
        TB[ThingsBoard Platform]
    end

    TOPIC_IN --> INT
    INT --> TB
    TB --> INT
    INT --> TOPIC_OUT
```

### Key Configuration

| Parameter | Description |
|-----------|-------------|
| Service URL | Pulsar broker URL |
| Topic | Topic to consume |
| Subscription | Subscription name |
| Subscription type | Exclusive/Shared/Failover |

### Connection Settings

```json
{
  "serviceUrl": "pulsar://pulsar.example.com:6650",
  "topic": "persistent://public/iot/telemetry",
  "subscriptionName": "thingsboard-sub",
  "subscriptionType": "Shared"
}
```

---

## Common Integration Patterns

### Event Sourcing

Capture all device events in a message stream:

```mermaid
graph TB
    DEVICES[Devices] --> TB[ThingsBoard]
    TB --> RE[Rule Engine]
    RE --> INT[Kafka Integration]
    INT --> KAFKA[Kafka Topic]
    KAFKA --> ANALYTICS[Analytics]
    KAFKA --> ARCHIVE[Data Lake]
```

### Message Replay

Re-process historical data:

```mermaid
graph LR
    ARCHIVE[Message Archive] --> KAFKA[Kafka]
    KAFKA --> INT[Integration]
    INT --> TB[ThingsBoard]
    TB --> REPROCESS[Reprocessed Data]
```

### Multi-System Integration

Bridge multiple messaging systems:

```mermaid
graph TB
    MQTT[MQTT Broker] --> INT1[MQTT Integration]
    KAFKA[Kafka] --> INT2[Kafka Integration]
    RABBIT[RabbitMQ] --> INT3[RabbitMQ Integration]

    INT1 --> TB[ThingsBoard]
    INT2 --> TB
    INT3 --> TB

    TB --> RE[Rule Engine]
    RE --> UNIFIED[Unified Data Model]
```

---

## Performance Tuning

### MQTT Integration

| Parameter | Recommendation |
|-----------|----------------|
| QoS | Use QoS 1 for balance |
| Keep Alive | 30-60 seconds |
| Max Inflight | Adjust based on throughput |
| Clean Session | false for persistence |

### Kafka Integration

| Parameter | Recommendation |
|-----------|----------------|
| Batch size | 16384-65536 bytes |
| Linger ms | 5-10 ms |
| Buffer memory | 32-64 MB |
| Compression | lz4 or snappy |
| Partitions | Scale with consumers |

### RabbitMQ Integration

| Parameter | Recommendation |
|-----------|----------------|
| Prefetch count | 10-50 messages |
| Acknowledgment | Manual with batching |
| Heartbeat | 60 seconds |
| Connection recovery | Enable automatic |

---

## Security Configuration

### TLS/SSL

All messaging integrations support encrypted connections:

```json
{
  "ssl": {
    "enabled": true,
    "trustStore": "/path/to/truststore.jks",
    "trustStorePassword": "secret",
    "keyStore": "/path/to/keystore.jks",
    "keyStorePassword": "secret"
  }
}
```

### Authentication Methods

| Platform | Methods |
|----------|---------|
| MQTT | Username/Password, Certificates |
| Kafka | SASL (PLAIN, SCRAM), SSL |
| RabbitMQ | Username/Password, Certificates |
| Pulsar | Token, TLS, OAuth2 |

---

## Common Pitfalls

### HTTP Integration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Webhook endpoint unreachable | No data received | Ensure ThingsBoard endpoint is publicly accessible |
| Missing security headers | Unauthorized requests accepted | Configure headers filter for authentication |
| Request timeout too short | Failed uplinks | Increase timeout for slow networks |
| Content-Type mismatch | Payload parsing fails | Ensure sender uses correct Content-Type header |
| Response status code ignored | Sender retries unnecessarily | Enable "Replace response status from 'No-Content' to 'OK'" |
| Synchronous downlink timing | Commands never delivered | Understand that downlinks require subsequent uplink |

### MQTT Integration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Client ID collision | Broker disconnects one client | Use unique client ID per integration |
| Topic filter too broad | Receiving irrelevant messages | Use specific topic patterns |
| QoS level mismatch | Message loss or duplication | Match QoS between broker and integration |
| Clean session enabled | Missed messages during reconnect | Disable clean session for persistence |
| TLS certificate validation | Connection rejected | Include full certificate chain |
| Last Will not configured | No disconnect notification | Configure LWT if needed for monitoring |

### Kafka Integration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Consumer group coordination | Uneven partition assignment | Use consistent group ID, monitor partition balance |
| Auto offset reset wrong | Missing or duplicate messages | Use "earliest" for replay, "latest" for real-time |
| Serialization format mismatch | Deserialization errors | Match serializer/deserializer configuration |
| Schema registry not configured | Avro/Protobuf parsing fails | Configure schema registry URL |
| Broker connection pool exhaustion | Timeouts under load | Increase connection limits |
| Offset commit failures | Message reprocessing | Monitor commit success rate |

### RabbitMQ Integration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Prefetch count too high | Memory exhaustion | Tune prefetch to match processing speed |
| Exchange type mismatch | Messages not routed | Match exchange type (direct, topic, fanout) |
| Queue not bound to exchange | Messages lost | Verify queue binding before sending |
| Dead letter queue not configured | Failed messages lost | Configure DLX for error handling |
| Connection recovery disabled | Integration stays disconnected | Enable automatic connection recovery |

### Apache Pulsar Integration

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Subscription type wrong | Message distribution issues | Use Shared for multiple consumers |
| Token authentication expired | Connection fails | Implement token refresh mechanism |
| Namespace not found | Subscription fails | Verify topic path format |

### General Messaging

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Message ordering assumptions | Out-of-order processing | Use timestamps from payload, not arrival |
| Poison messages | Consumer stuck | Implement error handling and dead letter |
| Backpressure not handled | Memory issues | Configure buffer limits and flow control |
| No retry logic | Transient failures cause data loss | Implement retry with exponential backoff |

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Connection refused | Wrong host/port | Verify broker address |
| Authentication failed | Invalid credentials | Check username/password |
| No messages received | Wrong topic/queue | Verify subscription |
| High latency | Network/batch settings | Tune buffer and batch sizes |
| Consumer lag | Slow processing | Increase consumers or partitions |
| Intermittent disconnects | Network instability | Enable reconnection, check keep-alive |
| Duplicate messages | At-least-once delivery | Implement deduplication logic |

## See Also

- [Cloud Integrations](./cloud-integrations.md) - AWS, Azure, GCP
- [LoRaWAN Integrations](./lorawan-integrations.md) - LPWAN networks
- [Message Queue](../08-message-queue/README.md) - Internal queue architecture
- [Rule Engine](../04-rule-engine/README.md) - Processing messages
