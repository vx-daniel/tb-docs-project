# External Nodes Reference

## Overview

External nodes integrate ThingsBoard with third-party systems by making HTTP requests, publishing to message brokers, calling cloud services, and sending notifications. They enable the rule engine to trigger actions in external systems based on IoT events.

## Node Quick Reference

### HTTP/REST

| Node | Class | Description |
|------|-------|-------------|
| REST API Call | TbRestApiCallNode | Call external REST API |

### Message Brokers

| Node | Class | Description |
|------|-------|-------------|
| Kafka | TbKafkaNode | Publish to Apache Kafka |
| MQTT | TbMqttNode | Publish to MQTT broker |
| RabbitMQ | TbRabbitMqNode | Publish to RabbitMQ |

### Cloud Services

| Node | Class | Description |
|------|-------|-------------|
| AWS SQS | TbSqsNode | Publish to AWS SQS |
| AWS SNS | TbSnsNode | Publish to AWS SNS |
| AWS Lambda | TbAwsLambdaNode | Invoke AWS Lambda |
| GCP Pub/Sub | TbPubSubNode | Publish to GCP Pub/Sub |
| Azure IoT Hub | TbAzureIotHubNode | Publish to Azure IoT Hub |

### Notifications

| Node | Class | Description |
|------|-------|-------------|
| Send Email | TbSendEmailNode | Send email via SMTP |
| Send SMS | TbSendSmsNode | Send SMS via provider |
| Send Notification | TbNotificationNode | Platform notifications |
| Send to Slack | TbSlackNode | Post to Slack channel |
| Twilio SMS | TbTwilioSmsNode | Send SMS via Twilio |
| Twilio Voice | TbTwilioVoiceNode | Make voice call via Twilio |

### AI Integration

| Node | Class | Description |
|------|-------|-------------|
| AI Request | TbAiNode | Send to AI model |

---

## REST API Call

Makes HTTP requests to external REST endpoints.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| restEndpointUrlPattern | string | URL pattern with variables |
| requestMethod | enum | GET, POST, PUT, DELETE, PATCH |
| headers | object | HTTP headers |
| readTimeoutMs | integer | Request timeout (ms) |
| maxParallelRequestsCount | integer | Max concurrent requests |
| useSimpleClientHttpFactory | boolean | Use simple HTTP client |
| useSystemProxyProperties | boolean | Use system proxy |
| parseToPlainText | boolean | Parse response as text |
| enableProxy | boolean | Enable proxy |
| proxyHost | string | Proxy hostname |
| proxyPort | integer | Proxy port |
| proxyUser | string | Proxy username |
| proxyPassword | string | Proxy password |
| proxyScheme | string | Proxy scheme (http/https) |

### URL Pattern Variables

Patterns support `${...}` syntax resolved from metadata and message:
- `${metadata.deviceId}` - Metadata value
- `${msg.sensorId}` - Message payload value

### Processing Flow

```mermaid
graph LR
    MSG[Message] --> URL[Resolve URL pattern]
    URL --> HEADERS[Add headers]
    HEADERS --> REQUEST[Make HTTP request]
    REQUEST -->|Success| SUCCESS[Success]
    REQUEST -->|Error| FAILURE[Failure]
```

### Response Metadata

| Key | Description |
|-----|-------------|
| status | HTTP status description |
| statusCode | HTTP status code (200, 404, etc.) |
| statusReason | Status reason phrase |
| headers | Response headers (JSON) |

### Example Configuration

```json
{
  "restEndpointUrlPattern": "https://api.example.com/devices/${metadata.deviceId}/data",
  "requestMethod": "POST",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer ${metadata.apiToken}"
  },
  "readTimeoutMs": 5000,
  "maxParallelRequestsCount": 10
}
```

---

## Kafka Node

Publishes messages to Apache Kafka topics.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| topicPattern | string | Topic pattern with variables |
| keyPattern | string | Message key pattern |
| bootstrapServers | string | Kafka broker addresses |
| acks | string | Acknowledgment mode (-1, 0, 1) |
| retries | integer | Retry count |
| batchSize | integer | Batch size (bytes) |
| linger | integer | Linger time (ms) |
| bufferMemory | integer | Buffer memory (bytes) |
| compression | string | Compression type |
| addMetadataKeyValuesAsKafkaHeaders | boolean | Add metadata as headers |
| otherProperties | object | Additional Kafka properties |

### Acknowledgment Modes

| Value | Behavior |
|-------|----------|
| 0 | Fire and forget |
| 1 | Leader acknowledgment |
| -1 (all) | All replicas acknowledgment |

### Response Metadata

| Key | Description |
|-----|-------------|
| offset | Kafka message offset |
| partition | Kafka partition number |
| topic | Kafka topic name |
| error | Error message (on failure) |

### Example Configuration

```json
{
  "topicPattern": "iot-telemetry-${metadata.deviceType}",
  "keyPattern": "${metadata.deviceId}",
  "bootstrapServers": "kafka1:9092,kafka2:9092",
  "acks": "-1",
  "retries": 3,
  "batchSize": 16384,
  "linger": 5,
  "bufferMemory": 33554432,
  "compression": "lz4",
  "addMetadataKeyValuesAsKafkaHeaders": true
}
```

### Kafka Headers

When `addMetadataKeyValuesAsKafkaHeaders` is enabled:
- Headers are prefixed with `tb_msg_md_`
- Example: `tb_msg_md_deviceName`, `tb_msg_md_deviceType`

---

## MQTT Node

Publishes messages to external MQTT brokers.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| topicPattern | string | Topic pattern with variables |
| host | string | MQTT broker hostname |
| port | integer | Broker port |
| connectTimeoutSec | integer | Connection timeout |
| clientId | string | MQTT client ID |
| appendClientIdSuffix | boolean | Add unique suffix to client ID |
| protocolVersion | enum | MQTT_3_1 or MQTT_5 |
| cleanSession | boolean | Clean session flag |
| ssl | boolean | Enable SSL/TLS |
| retainedMessage | boolean | Publish as retained |
| parseToPlainText | boolean | Send as plain text |
| credentials | object | Authentication credentials |

### Credential Types

| Type | Fields |
|------|--------|
| anonymous | None |
| basic | username, password |
| certificate | certPemBlock, privateKeyPemBlock |

### Protocol Versions

| Version | Max Client ID | Features |
|---------|---------------|----------|
| MQTT_3_1 | 23 chars | Standard MQTT |
| MQTT_5 | 256 chars | Enhanced features |

### Example Configuration

```json
{
  "topicPattern": "devices/${metadata.deviceId}/telemetry",
  "host": "mqtt.example.com",
  "port": 8883,
  "connectTimeoutSec": 10,
  "clientId": "thingsboard-rule-engine",
  "appendClientIdSuffix": true,
  "protocolVersion": "MQTT_3_1",
  "cleanSession": true,
  "ssl": true,
  "retainedMessage": false,
  "credentials": {
    "type": "basic",
    "username": "publisher",
    "password": "secret"
  }
}
```

---

## RabbitMQ Node

Publishes messages to RabbitMQ.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| exchangeNamePattern | string | Exchange name pattern |
| routingKeyPattern | string | Routing key pattern |
| messageProperties | object | AMQP message properties |
| host | string | RabbitMQ hostname |
| port | integer | AMQP port |
| virtualHost | string | Virtual host |
| username | string | Username |
| password | string | Password |
| automaticRecoveryEnabled | boolean | Enable auto-recovery |
| connectionTimeout | integer | Connection timeout (ms) |

### Example Configuration

```json
{
  "exchangeNamePattern": "iot-exchange",
  "routingKeyPattern": "telemetry.${metadata.deviceType}",
  "host": "rabbitmq.example.com",
  "port": 5672,
  "virtualHost": "/",
  "username": "thingsboard",
  "password": "secret",
  "automaticRecoveryEnabled": true,
  "connectionTimeout": 5000
}
```

---

## AWS SQS Node

Publishes messages to Amazon Simple Queue Service.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| queueType | enum | STANDARD or FIFO |
| queueUrlPattern | string | SQS queue URL pattern |
| delaySeconds | integer | Message delay |
| messageGroupIdPattern | string | FIFO group ID pattern |
| accessKeyId | string | AWS access key |
| secretAccessKey | string | AWS secret key |
| region | string | AWS region |

### Example Configuration

```json
{
  "queueType": "STANDARD",
  "queueUrlPattern": "https://sqs.us-east-1.amazonaws.com/123456789/iot-queue",
  "delaySeconds": 0,
  "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
  "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "region": "us-east-1"
}
```

---

## AWS SNS Node

Publishes messages to Amazon Simple Notification Service.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| topicArnPattern | string | SNS topic ARN pattern |
| accessKeyId | string | AWS access key |
| secretAccessKey | string | AWS secret key |
| region | string | AWS region |

### Example Configuration

```json
{
  "topicArnPattern": "arn:aws:sns:us-east-1:123456789:iot-alerts",
  "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
  "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "region": "us-east-1"
}
```

---

## AWS Lambda Node

Invokes AWS Lambda functions.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| functionNamePattern | string | Lambda function name |
| qualifier | string | Function version/alias |
| accessKeyId | string | AWS access key |
| secretAccessKey | string | AWS secret key |
| region | string | AWS region |

### Example Configuration

```json
{
  "functionNamePattern": "process-iot-data",
  "qualifier": "$LATEST",
  "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
  "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "region": "us-east-1"
}
```

---

## GCP Pub/Sub Node

Publishes messages to Google Cloud Pub/Sub.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| projectId | string | GCP project ID |
| topicName | string | Pub/Sub topic name |
| serviceAccountKey | string | Service account JSON key |
| messageAttributes | object | Message attributes |

### Example Configuration

```json
{
  "projectId": "my-gcp-project",
  "topicName": "iot-telemetry",
  "serviceAccountKey": "{...service account JSON...}",
  "messageAttributes": {
    "deviceType": "${metadata.deviceType}"
  }
}
```

---

## Azure IoT Hub Node

Sends device-to-cloud messages to Azure IoT Hub.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| topicPattern | string | Topic pattern |
| hostname | string | IoT Hub hostname |
| credentials | object | Authentication credentials |

### Example Configuration

```json
{
  "topicPattern": "devices/${metadata.deviceId}/messages/events/",
  "hostname": "my-iot-hub.azure-devices.net",
  "credentials": {
    "type": "sas",
    "sasKey": "..."
  }
}
```

---

## Send Email Node

Sends email via SMTP server.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| useSystemSmtpSettings | boolean | Use platform SMTP config |
| smtpHost | string | SMTP server hostname |
| smtpPort | integer | SMTP port |
| smtpProtocol | enum | SMTP, SMTPS, TLS |
| username | string | SMTP username |
| password | string | SMTP password |
| timeout | integer | Connection timeout (ms) |

### Expected Message Format

The node expects a message formatted by the "To Email" transformation node:

```json
{
  "from": "alerts@company.com",
  "to": "admin@company.com",
  "cc": "",
  "bcc": "",
  "subject": "Alert Notification",
  "body": "Alert message body",
  "html": false
}
```

### Processing Flow

```mermaid
graph LR
    EMAIL[Email Message] --> SMTP{SMTP Config}
    SMTP -->|System| SYS[Use system settings]
    SMTP -->|Custom| CUSTOM[Use node settings]
    SYS --> SEND[Send email]
    CUSTOM --> SEND
    SEND --> SUCCESS[Success]
```

---

## Send SMS Node

Sends SMS via configured SMS provider.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| useSystemSmsSettings | boolean | Use platform SMS config |
| smsProviderConfigurations | object | Custom SMS provider config |
| numbersToTemplate | string | Recipient phone number pattern |

### Expected Message Format

```json
{
  "to": "+1234567890",
  "message": "Alert: Temperature exceeded threshold"
}
```

---

## Send Notification Node

Sends notifications through the ThingsBoard notification system.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| templateId | UUID | Notification template ID |
| targets | object | Notification targets |

### Target Types

- User by ID
- User group
- Customer users
- Tenant administrators
- Affected users (alarm-related)

---

## Send to Slack Node

Posts messages to Slack channels.

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| webhookUrl | string | Slack webhook URL |
| messageTemplate | string | Message template |

### Example Configuration

```json
{
  "webhookUrl": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX",
  "messageTemplate": "Alert from ${metadata.deviceName}: Temperature is ${msg.temperature}°C"
}
```

---

## AI Request Node

Sends requests to AI models (e.g., OpenAI, Azure OpenAI).

### Configuration

| Field | Type | Description |
|-------|------|-------------|
| modelProvider | enum | AI provider |
| modelName | string | Model name |
| promptTemplate | string | Prompt template |
| maxTokens | integer | Maximum response tokens |
| temperature | number | Creativity parameter |

### Example Configuration

```json
{
  "modelProvider": "OPENAI",
  "modelName": "gpt-4",
  "promptTemplate": "Analyze this sensor data and identify anomalies: ${msg}",
  "maxTokens": 500,
  "temperature": 0.7
}
```

---

## Common Patterns

### Webhook Integration Pattern

```mermaid
graph LR
    EVENT[Event] --> TRANSFORM[Transform to JSON]
    TRANSFORM --> REST[REST API Call]
    REST -->|Success| LOG[Log Success]
    REST -->|Failure| RETRY[Retry Queue]
```

### Multi-Channel Alert Pattern

```mermaid
graph TB
    ALARM[Alarm Created] --> EMAIL[Send Email]
    ALARM --> SMS[Send SMS]
    ALARM --> SLACK[Send to Slack]
    EMAIL --> LOG[Log]
    SMS --> LOG
    SLACK --> LOG
```

### Event Streaming Pattern

```mermaid
graph LR
    TEL[Telemetry] --> SAVE[Save to DB]
    SAVE --> KAFKA[Kafka Node]
    KAFKA --> ANALYTICS[External Analytics]
```

### Serverless Processing Pattern

```mermaid
graph LR
    DATA[Device Data] --> LAMBDA[AWS Lambda]
    LAMBDA -->|Processed| TRANSFORM[Transform Result]
    TRANSFORM --> SAVE[Save Telemetry]
```

## Best Practices

1. **Handle failures gracefully** - Always connect Failure output
2. **Use timeouts** - Set appropriate timeouts for external calls
3. **Limit parallelism** - Control concurrent requests to avoid overload
4. **Secure credentials** - Use metadata or attributes for API keys
5. **Monitor external calls** - Track latency and error rates
6. **Batch when possible** - Reduce external call frequency

## See Also

- [Action Nodes](./action-nodes.md) - Platform actions
- [Transformation Nodes](./transformation-nodes.md) - Prepare messages
- [Integrations](../../14-integrations/README.md) - Platform integrations
- [Node Categories](../node-categories.md) - All node categories
