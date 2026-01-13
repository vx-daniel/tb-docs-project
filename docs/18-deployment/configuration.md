# Configuration

## Overview

ThingsBoard is configured primarily through environment variables, making it easy to manage across different deployment methods. This guide covers key configuration parameters organized by functional area.

## Configuration Methods

### Environment Variables (Recommended)

Environment variables are the recommended configuration method:

| Deployment | How to Set |
|------------|------------|
| Linux | `/usr/share/thingsboard/conf/thingsboard.conf` |
| Docker | `-e VAR=value` or docker-compose environment |
| Kubernetes | ConfigMaps, Secrets, env in deployment |
| Windows | `thingsboard.yml` in conf directory |

### Linux Configuration

```bash
# Edit configuration file
sudo nano /usr/share/thingsboard/conf/thingsboard.conf

# Add environment variables
export HTTP_BIND_PORT=8080
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/thingsboard
```

### Docker Configuration

```bash
docker run -e HTTP_BIND_PORT=8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/thingsboard \
  thingsboard/tb-postgres
```

### Kubernetes ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tb-config
data:
  HTTP_BIND_PORT: "8080"
  TB_QUEUE_TYPE: "kafka"
  TB_KAFKA_SERVERS: "kafka:9092"
```

## Server Configuration

### HTTP Server

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| server.address | HTTP_BIND_ADDRESS | 0.0.0.0 | Bind address |
| server.port | HTTP_BIND_PORT | 8080 | HTTP port |
| server.http2.enabled | HTTP2_ENABLED | true | HTTP/2 support |

### SSL/TLS

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| server.ssl.enabled | SSL_ENABLED | false | Enable SSL |
| server.ssl.credentials.type | SSL_CREDENTIALS_TYPE | PEM | PEM or KEYSTORE |
| server.ssl.credentials.pem.cert_file | SSL_PEM_CERT | server.pem | Certificate file |
| server.ssl.credentials.pem.key_file | SSL_PEM_KEY | server_key.pem | Private key file |

### WebSocket

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| server.ws.send_timeout | TB_SERVER_WS_SEND_TIMEOUT | 5000 | Send timeout (ms) |
| server.ws.ping_timeout | TB_SERVER_WS_PING_TIMEOUT | 30000 | Ping timeout (ms) |
| server.ws.max_queue_messages_per_session | TB_SERVER_WS_DEFAULT_QUEUE_MESSAGES_PER_SESSION | 1000 | Max queue size |

## Database Configuration

### PostgreSQL

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| spring.datasource.url | SPRING_DATASOURCE_URL | jdbc:postgresql://localhost:5432/thingsboard | JDBC URL |
| spring.datasource.username | SPRING_DATASOURCE_USERNAME | thingsboard | Username |
| spring.datasource.password | SPRING_DATASOURCE_PASSWORD | postgres | Password |
| spring.datasource.hikari.maximumPoolSize | SPRING_DATASOURCE_MAXIMUM_POOL_SIZE | 16 | Connection pool size |

### Cassandra

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| cassandra.url | CASSANDRA_URL | localhost:9042 | Contact points |
| cassandra.keyspace_name | CASSANDRA_KEYSPACE_NAME | thingsboard | Keyspace |
| cassandra.username | CASSANDRA_USERNAME | - | Username |
| cassandra.password | CASSANDRA_PASSWORD | - | Password |

### Redis Cache

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| cache.type | CACHE_TYPE | caffeine | Cache type (caffeine, redis) |
| redis.connection.type | REDIS_CONNECTION_TYPE | standalone | standalone, cluster, sentinel |
| redis.standalone.host | REDIS_HOST | localhost | Redis host |
| redis.standalone.port | REDIS_PORT | 6379 | Redis port |

## Message Queue Configuration

### Queue Type Selection

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| queue.type | TB_QUEUE_TYPE | in-memory | in-memory, kafka, aws-sqs, pubsub, service-bus, rabbitmq |

### Kafka Configuration

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| queue.kafka.bootstrap.servers | TB_KAFKA_SERVERS | localhost:9092 | Kafka brokers |
| queue.kafka.replication_factor | TB_QUEUE_KAFKA_REPLICATION_FACTOR | 1 | Replication factor |
| queue.kafka.acks | TB_KAFKA_ACKS | all | Acknowledgment level |

### Kafka Topic Settings

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| queue.kafka.topic-properties.core.partitions | TB_QUEUE_CORE_PARTITIONS | 10 | Core topic partitions |
| queue.kafka.topic-properties.rule-engine.partitions | TB_QUEUE_RULE_ENGINE_PARTITIONS | 10 | Rule engine partitions |
| queue.kafka.topic-properties.transport-api.partitions | TB_QUEUE_TRANSPORT_API_PARTITIONS | 10 | Transport partitions |

## Transport Configuration

### MQTT Transport

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| transport.mqtt.bind_address | MQTT_BIND_ADDRESS | 0.0.0.0 | Bind address |
| transport.mqtt.bind_port | MQTT_BIND_PORT | 1883 | MQTT port |
| transport.mqtt.ssl.enabled | MQTT_SSL_ENABLED | false | Enable SSL |
| transport.mqtt.ssl.bind_port | MQTT_SSL_BIND_PORT | 8883 | MQTTS port |
| transport.mqtt.max_payload_size | MQTT_MAX_PAYLOAD_SIZE | 65536 | Max message size |

### HTTP Transport

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| transport.http.request_timeout | HTTP_REQUEST_TIMEOUT | 60000 | Request timeout (ms) |
| transport.http.max_request_size | HTTP_MAX_REQUEST_SIZE | 65536 | Max request size |

### CoAP Transport

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| transport.coap.bind_address | COAP_BIND_ADDRESS | 0.0.0.0 | Bind address |
| transport.coap.bind_port | COAP_BIND_PORT | 5683 | CoAP port |
| transport.coap.dtls.bind_port | COAP_DTLS_BIND_PORT | 5684 | DTLS port |

### LwM2M Transport

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| transport.lwm2m.enabled | LWM2M_ENABLED | true | Enable LwM2M |
| transport.lwm2m.server.bind_port | LWM2M_SERVER_PORT | 5685 | Server port |
| transport.lwm2m.server.bind_port_bs | LWM2M_BS_PORT | 5687 | Bootstrap port |

## Security Configuration

### JWT Settings

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| security.jwt.tokenExpirationTime | JWT_TOKEN_EXPIRATION_TIME | 9000 | Token lifetime (sec) |
| security.jwt.refreshTokenExpTime | JWT_REFRESH_TOKEN_EXPIRATION_TIME | 604800 | Refresh token lifetime |
| security.jwt.tokenSigningKey | JWT_TOKEN_SIGNING_KEY | - | Signing key |

### Rate Limiting

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| security.rateLimits.tenant.enabled | TB_RATE_LIMITS_TENANT_ENABLED | false | Enable tenant rate limits |
| security.rateLimits.tenant.config | TB_RATE_LIMITS_TENANT_CONFIGURATION | 1000:1,20000:60 | Rate limit config |

### Password Policy

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| security.user.password.min_length | SECURITY_PASSWORD_MIN_LENGTH | 6 | Minimum length |
| security.user.password.max_length | SECURITY_PASSWORD_MAX_LENGTH | 72 | Maximum length |

## Actor System Configuration

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| actors.tenant.create_components_on_init | ACTORS_TENANT_CREATE_COMPONENTS_ON_INIT | true | Create on init |
| actors.session.max_concurrent_sessions_per_device | ACTORS_MAX_CONCURRENT_SESSIONS_PER_DEVICE | 1 | Sessions per device |
| actors.rule.chain.debug_mode_rate_limits | TB_RULE_CHAIN_DEBUG_MODE_RATE_LIMITS | 1000:1,10000:60 | Debug rate limits |

## Rule Engine Configuration

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| queue.rule-engine.topic | TB_QUEUE_RULE_ENGINE_TOPIC | tb_rule_engine | Topic name |
| queue.rule-engine.poll-interval | TB_QUEUE_RULE_ENGINE_POLL_INTERVAL_MS | 25 | Poll interval (ms) |
| queue.rule-engine.pack-processing-timeout | TB_QUEUE_RULE_ENGINE_PACK_PROCESSING_TIMEOUT_MS | 2000 | Processing timeout |

## Service Discovery

### Zookeeper

| Parameter | Environment Variable | Default | Description |
|-----------|---------------------|---------|-------------|
| zk.enabled | ZOOKEEPER_ENABLED | false | Enable Zookeeper |
| zk.url | ZOOKEEPER_URL | localhost:2181 | Zookeeper URL |

## Logging Configuration

### Log Levels

```bash
# Set log level via environment
export LOGGING_LEVEL_ORG_THINGSBOARD=INFO
export LOGGING_LEVEL_ROOT=WARN
```

| Environment Variable | Description |
|---------------------|-------------|
| LOGGING_LEVEL_ROOT | Root logger level |
| LOGGING_LEVEL_ORG_THINGSBOARD | ThingsBoard logger |
| LOGGING_LEVEL_ORG_THINGSBOARD_SERVER_TRANSPORT | Transport logger |
| LOGGING_LEVEL_ORG_THINGSBOARD_RULE_ENGINE | Rule engine logger |

## Example Configurations

### Development Environment

```bash
export HTTP_BIND_PORT=8080
export TB_QUEUE_TYPE=in-memory
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/thingsboard
export SPRING_DATASOURCE_USERNAME=thingsboard
export SPRING_DATASOURCE_PASSWORD=thingsboard
export JWT_TOKEN_SIGNING_KEY=mySecretKey123
```

### Production Environment

```bash
export HTTP_BIND_PORT=8080
export SSL_ENABLED=true
export SSL_PEM_CERT=/etc/ssl/certs/server.pem
export SSL_PEM_KEY=/etc/ssl/private/server_key.pem
export TB_QUEUE_TYPE=kafka
export TB_KAFKA_SERVERS=kafka1:9092,kafka2:9092,kafka3:9092
export SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-primary:5432/thingsboard
export SPRING_DATASOURCE_USERNAME=thingsboard
export SPRING_DATASOURCE_PASSWORD=<secure-password>
export CACHE_TYPE=redis
export REDIS_CONNECTION_TYPE=cluster
export REDIS_CLUSTER_NODES=redis1:6379,redis2:6379,redis3:6379
export JWT_TOKEN_SIGNING_KEY=<secure-signing-key>
```

### Docker Compose Example

```yaml
services:
  thingsboard:
    environment:
      - TB_QUEUE_TYPE=kafka
      - TB_KAFKA_SERVERS=kafka:9092
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/thingsboard
      - SPRING_DATASOURCE_USERNAME=thingsboard
      - SPRING_DATASOURCE_PASSWORD=${POSTGRES_PASSWORD}
      - CACHE_TYPE=redis
      - REDIS_HOST=redis
      - JWT_TOKEN_SIGNING_KEY=${JWT_KEY}
```

### Kubernetes ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thingsboard-config
data:
  TB_QUEUE_TYPE: kafka
  TB_KAFKA_SERVERS: kafka-headless:9092
  CACHE_TYPE: redis
  REDIS_HOST: redis-master
  HTTP_BIND_PORT: "8080"
---
apiVersion: v1
kind: Secret
metadata:
  name: thingsboard-secrets
type: Opaque
stringData:
  SPRING_DATASOURCE_PASSWORD: secure-password
  JWT_TOKEN_SIGNING_KEY: secure-jwt-key
```

## Configuration Best Practices

| Practice | Description |
|----------|-------------|
| Use environment variables | Easier to manage across deployments |
| Externalize secrets | Use secret management tools |
| Version control | Track configuration changes |
| Environment separation | Different configs per environment |
| Validate on startup | Check critical parameters |

## See Also

- [Installation Options](./installation-options.md) - Deployment methods
- [Monitoring](./monitoring-operations.md) - Operations
- [Message Queue](../08-message-queue/README.md) - Queue configuration
- [Security](../09-security/README.md) - Security settings
