# REST API Overview

## Overview

The REST API provides HTTP endpoints for managing entities, retrieving telemetry, configuring rule chains, and administering the platform. All endpoints follow RESTful conventions with JSON payloads. Authentication uses JWT tokens, and authorization is role-based with tenant isolation. The API supports pagination, filtering, and sorting for list operations.

## Architecture

```mermaid
graph TB
    subgraph "Clients"
        WEB[Web UI]
        MOBILE[Mobile Apps]
        EXT[External Systems]
    end

    subgraph "API Layer"
        AUTH[Auth Controller]
        ENTITY[Entity Controllers]
        TEL[Telemetry Controller]
        RULE[Rule Chain Controller]
        ADMIN[Admin Controller]
    end

    subgraph "Base Infrastructure"
        BASE[BaseController]
        ACCESS[AccessControlService]
        VALID[Validators]
        ERROR[Error Handler]
    end

    subgraph "Services"
        SVC[Business Services]
        DAO[Data Access]
    end

    WEB --> AUTH
    MOBILE --> AUTH
    EXT --> AUTH

    AUTH --> BASE
    ENTITY --> BASE
    TEL --> BASE
    RULE --> BASE
    ADMIN --> BASE

    BASE --> ACCESS
    BASE --> VALID
    BASE --> ERROR
    BASE --> SVC
    SVC --> DAO
```

## Base URL

All API endpoints are prefixed with `/api`:

```
https://{host}:{port}/api/{endpoint}
```

| Environment | Base URL |
|-------------|----------|
| Local Development | `http://localhost:8080/api` |
| Production | `https://your-domain.com/api` |

## Authentication

### JWT Token Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth API
    participant T as Token Service
    participant DB as Database

    C->>A: POST /api/auth/login
    Note over C,A: {username, password}

    A->>DB: Validate credentials
    DB-->>A: User + Credentials

    A->>T: Generate tokens
    T-->>A: JWT Pair

    A-->>C: {token, refreshToken}

    Note over C: Store tokens

    C->>A: GET /api/device/{id}
    Note over C,A: Authorization: Bearer {token}

    A->>T: Validate token
    T-->>A: SecurityUser

    A-->>C: Device data
```

### Token Structure

| Field | Description |
|-------|-------------|
| `token` | Short-lived access token (default: 2.5 hours) |
| `refreshToken` | Long-lived refresh token (default: 1 week) |

### Authentication Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/auth/login` | POST | Authenticate and get tokens |
| `/api/auth/token` | POST | Refresh access token |
| `/api/auth/logout` | POST | Record logout (audit only) |
| `/api/auth/user` | GET | Get current user info |
| `/api/auth/changePassword` | POST | Change password |

### Login Request

```json
{
  "username": "tenant@thingsboard.org",
  "password": "tenant"
}
```

### Login Response

```json
{
  "token": "eyJhbGciOiJIUzUxMiJ9...",
  "refreshToken": "eyJhbGciOiJIUzUxMiJ9..."
}
```

### Using the Token

Include the access token in the Authorization header:

```
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

## User Roles and Authorities

```mermaid
graph TB
    subgraph "Authorities"
        SYS[SYS_ADMIN]
        TENANT[TENANT_ADMIN]
        CUSTOMER[CUSTOMER_USER]
    end

    subgraph "Resources"
        TENANTS[Tenants]
        DEVICES[Devices]
        ASSETS[Assets]
        DASHBOARDS[Dashboards]
        RULES[Rule Chains]
        USERS[Users]
    end

    SYS --> TENANTS
    SYS --> USERS

    TENANT --> DEVICES
    TENANT --> ASSETS
    TENANT --> DASHBOARDS
    TENANT --> RULES
    TENANT --> USERS

    CUSTOMER --> DEVICES
    CUSTOMER --> ASSETS
    CUSTOMER --> DASHBOARDS
```

| Authority | Description | Scope |
|-----------|-------------|-------|
| SYS_ADMIN | System administrator | Platform-wide |
| TENANT_ADMIN | Tenant administrator | Single tenant |
| CUSTOMER_USER | Customer user | Assigned entities only |

## Controllers Overview

The platform provides **57 REST controllers** organized by domain:

### Core Entity Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| DeviceController | `/api/device` | Device CRUD, credentials |
| AssetController | `/api/asset` | Asset management |
| CustomerController | `/api/customer` | Customer management |
| UserController | `/api/user` | User management |
| TenantController | `/api/tenant` | Tenant management (SYS_ADMIN) |
| DashboardController | `/api/dashboard` | Dashboard management |

### Data Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| TelemetryController | `/api/plugins/telemetry` | Telemetry and attributes |
| AlarmController | `/api/alarm` | Alarm management |
| EventController | `/api/event` | System events |
| AuditLogController | `/api/audit/logs` | Audit trail |

### Configuration Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| RuleChainController | `/api/ruleChain` | Rule chain management |
| DeviceProfileController | `/api/deviceProfile` | Device profiles |
| AssetProfileController | `/api/assetProfile` | Asset profiles |
| WidgetsBundleController | `/api/widgetsBundle` | Widget bundles |
| WidgetTypeController | `/api/widgetType` | Widget types |

### Integration Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| EntityRelationController | `/api/relation` | Entity relations |
| EntityQueryController | `/api/entitiesQuery` | Advanced queries |
| RpcV1Controller | `/api/plugins/rpc` | RPC commands (deprecated) |
| RpcV2Controller | `/api/rpc` | RPC commands (current) |
| EdgeController | `/api/edge` | Edge management |

### Additional Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| ApiKeyController | `/api/apiKey` | API key management |
| AiModelController | `/api/aiModel` | AI model configuration |
| JobController | `/api/job` | Background job management |
| NotificationController | `/api/notification` | Notification management |
| NotificationRuleController | `/api/notification/rule` | Notification rules |
| NotificationTemplateController | `/api/notification/template` | Notification templates |
| NotificationTargetController | `/api/notification/target` | Notification targets |
| OtaPackageController | `/api/otaPackage` | Firmware/software updates |
| QueueController | `/api/queue` | Queue configuration |
| QueueStatsController | `/api/queueStats` | Queue statistics |
| ImageController | `/api/image` | Image management |
| Lwm2mController | `/api/lwm2m` | LwM2M device configuration |
| SystemInfoController | `/api/systemInfo` | System information |

### Version Control Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| EntitiesVersionControlController | `/api/entities/vc` | Git-based entity version control |
| AutoCommitController | `/api/autoCommit` | Auto-commit settings |

### Device Connectivity Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| DeviceConnectivityController | `/api/device-connectivity` | Protocol-specific connection commands |

### Mobile App Controllers

| Controller | Base Path | Description |
|------------|-----------|-------------|
| MobileAppController | `/api/mobileApp` | Mobile application management |
| MobileAppBundleController | `/api/mobileAppBundle` | Mobile app bundles |

## Common Patterns

### Entity ID Format

All entity IDs are time-based UUIDs:

```
784f394c-42b6-435a-983c-b7beff2784f9
```

### Standard CRUD Operations

```mermaid
graph LR
    subgraph "CRUD Pattern"
        CREATE[POST /api/device]
        READ[GET /api/device/{id}]
        UPDATE[POST /api/device]
        DELETE[DELETE /api/device/{id}]
        LIST[GET /api/tenant/devices]
    end
```

| Operation | HTTP Method | URL Pattern | Body |
|-----------|-------------|-------------|------|
| Create | POST | `/api/{entity}` | Entity JSON |
| Read | GET | `/api/{entity}/{id}` | - |
| Update | POST | `/api/{entity}` | Entity JSON with ID |
| Delete | DELETE | `/api/{entity}/{id}` | - |
| List | GET | `/api/{scope}/{entities}` | - |

### Pagination

List endpoints support pagination with these parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| pageSize | int | 10 | Items per page (max 1000) |
| page | int | 0 | Page number (0-indexed) |
| textSearch | string | - | Case-insensitive substring filter |
| sortProperty | string | - | Field to sort by |
| sortOrder | string | ASC | Sort direction (ASC/DESC) |

**Request:**
```
GET /api/tenant/devices?pageSize=20&page=0&sortProperty=name&sortOrder=ASC&textSearch=sensor
```

**Response:**
```json
{
  "data": [
    {"id": {"id": "..."}, "name": "Sensor-001", ...},
    {"id": {"id": "..."}, "name": "Sensor-002", ...}
  ],
  "totalPages": 5,
  "totalElements": 100,
  "hasNext": true
}
```

### Time-Based Pagination

Some endpoints support time-based filtering:

| Parameter | Type | Description |
|-----------|------|-------------|
| startTime | long | Start timestamp (ms) |
| endTime | long | End timestamp (ms) |

### Entity Response Structure

All entities follow a consistent structure:

```json
{
  "id": {
    "entityType": "DEVICE",
    "id": "784f394c-42b6-435a-983c-b7beff2784f9"
  },
  "createdTime": 1609459200000,
  "tenantId": {
    "entityType": "TENANT",
    "id": "..."
  },
  "name": "Temperature Sensor",
  "type": "sensor",
  "label": "Building A - Floor 1",
  "additionalInfo": {}
}
```

## Error Handling

### Error Response Structure

```json
{
  "status": 400,
  "message": "Parameter 'deviceId' can't be empty!",
  "errorCode": 31,
  "timestamp": "2024-01-15T10:30:00.000+00:00"
}
```

### Error Codes

| Code | Name | HTTP Status | Description |
|------|------|-------------|-------------|
| 2 | GENERAL | 500 | Internal server error |
| 10 | AUTHENTICATION | 401 | Authentication required |
| 11 | JWT_TOKEN_EXPIRED | 401 | Token has expired |
| 15 | CREDENTIALS_EXPIRED | 401 | Password expired |
| 20 | PERMISSION_DENIED | 403 | Insufficient permissions |
| 30 | INVALID_ARGUMENTS | 400 | Invalid request arguments |
| 31 | BAD_REQUEST_PARAMS | 400 | Bad request parameters |
| 32 | ITEM_NOT_FOUND | 404 | Entity not found |
| 33 | TOO_MANY_REQUESTS | 429 | Rate limit exceeded |
| 34 | TOO_MANY_UPDATES | 429 | Update rate limit exceeded |
| 35 | VERSION_CONFLICT | 409 | Optimistic lock failure |
| 40 | SUBSCRIPTION_VIOLATION | 403 | License/subscription limit |
| 41 | ENTITIES_LIMIT_EXCEEDED | 403 | Entity count limit exceeded |
| 45 | PASSWORD_VIOLATION | 401 | Password policy violation |
| 46 | DATABASE | 500 | Database error |

### Specialized Error Responses

**Credentials Expired Response:**
```json
{
  "status": 401,
  "message": "User credentials have expired",
  "errorCode": 15,
  "resetToken": "eyJhbGciOiJIUzI1NiJ9...",
  "timestamp": 1673913602547
}
```

**Entities Limit Exceeded Response:**
```json
{
  "status": 403,
  "message": "Devices limit reached",
  "errorCode": 41,
  "entityType": "DEVICE",
  "limit": 1000,
  "timestamp": 1673913602547
}
```

### HTTP Status Codes

| Status | Meaning | When Used |
|--------|---------|-----------|
| 200 | OK | Successful read/update |
| 201 | Created | Successful create |
| 400 | Bad Request | Invalid parameters |
| 401 | Unauthorized | Missing/invalid token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Entity doesn't exist |
| 409 | Conflict | Version conflict |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

## Device API Examples

### Create Device

```http
POST /api/device HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "Temperature Sensor 001",
  "type": "temperature-sensor",
  "label": "Building A - Room 101",
  "deviceProfileId": {
    "entityType": "DEVICE_PROFILE",
    "id": "profile-uuid"
  },
  "additionalInfo": {
    "description": "Main temperature sensor"
  }
}
```

### Get Device

```http
GET /api/device/784f394c-42b6-435a-983c-b7beff2784f9 HTTP/1.1
Authorization: Bearer {token}
```

### List Tenant Devices

```http
GET /api/tenant/devices?pageSize=20&page=0&type=temperature-sensor HTTP/1.1
Authorization: Bearer {token}
```

### Get Device Credentials

```http
GET /api/device/784f394c-42b6-435a-983c-b7beff2784f9/credentials HTTP/1.1
Authorization: Bearer {token}
```

**Response:**
```json
{
  "id": {"id": "..."},
  "deviceId": {"id": "784f394c-..."},
  "credentialsType": "ACCESS_TOKEN",
  "credentialsId": "A1_TEST_TOKEN"
}
```

## Telemetry API Examples

### Get Latest Telemetry

```http
GET /api/plugins/telemetry/DEVICE/784f394c-42b6-435a-983c-b7beff2784f9/values/timeseries?keys=temperature,humidity HTTP/1.1
Authorization: Bearer {token}
```

**Response:**
```json
{
  "temperature": [{"ts": 1609459200000, "value": "25.5"}],
  "humidity": [{"ts": 1609459200000, "value": "60"}]
}
```

### Get Historical Telemetry

```http
GET /api/plugins/telemetry/DEVICE/{deviceId}/values/timeseries?keys=temperature&startTs=1609459200000&endTs=1609545600000&interval=3600000&agg=AVG HTTP/1.1
Authorization: Bearer {token}
```

**Aggregation Parameters:**

| Parameter | Description |
|-----------|-------------|
| startTs | Start timestamp (ms) |
| endTs | End timestamp (ms) |
| interval | Aggregation interval (ms) |
| agg | Aggregation: NONE, MIN, MAX, AVG, SUM, COUNT |
| limit | Max data points |

### Save Telemetry (Server-Side)

```http
POST /api/plugins/telemetry/DEVICE/{deviceId}/timeseries/ANY HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}

{
  "temperature": 25.5,
  "humidity": 60
}
```

### Get Attributes

```http
GET /api/plugins/telemetry/DEVICE/{deviceId}/values/attributes?keys=firmware,config HTTP/1.1
Authorization: Bearer {token}
```

### Save Attributes

```http
POST /api/plugins/telemetry/DEVICE/{deviceId}/attributes/SERVER_SCOPE HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}

{
  "threshold": 30,
  "alertEnabled": true
}
```

**Attribute Scopes:**

| Scope | Description | Writable By |
|-------|-------------|-------------|
| SERVER_SCOPE | Server attributes | Server/API |
| SHARED_SCOPE | Shared attributes | Server/API |
| CLIENT_SCOPE | Client attributes | Device |

## Alarm API Examples

### Get Alarms

```http
GET /api/alarm/DEVICE/{deviceId}?pageSize=20&page=0&status=ACTIVE_UNACK HTTP/1.1
Authorization: Bearer {token}
```

### Acknowledge Alarm

```http
POST /api/alarm/{alarmId}/ack HTTP/1.1
Authorization: Bearer {token}
```

### Clear Alarm

```http
POST /api/alarm/{alarmId}/clear HTTP/1.1
Authorization: Bearer {token}
```

## RPC API Examples

### Send One-Way RPC

```http
POST /api/rpc/oneway/{deviceId} HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}

{
  "method": "setValue",
  "params": {"pin": 1, "value": true}
}
```

### Send Two-Way RPC

```http
POST /api/rpc/twoway/{deviceId} HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}

{
  "method": "getStatus",
  "params": {}
}
```

**Response:**
```json
{
  "result": {"status": "OK", "uptime": 3600}
}
```

## Access Control

### Permission Check Flow

```mermaid
graph TB
    REQ[API Request] --> AUTH{Authenticated?}
    AUTH -->|No| DENY1[401 Unauthorized]
    AUTH -->|Yes| ROLE{Has Authority?}

    ROLE -->|No| DENY2[403 Forbidden]
    ROLE -->|Yes| TENANT{Same Tenant?}

    TENANT -->|No| DENY3[403 Forbidden]
    TENANT -->|Yes| ENTITY{Entity Access?}

    ENTITY -->|No| DENY4[404 Not Found]
    ENTITY -->|Yes| OK[Process Request]
```

### Entity Access Rules

| Authority | Own Tenant | Assigned Entities | All Entities |
|-----------|------------|-------------------|--------------|
| SYS_ADMIN | N/A | N/A | All tenants |
| TENANT_ADMIN | Full access | N/A | Own tenant only |
| CUSTOMER_USER | Read only | Full access | Assigned only |

### Pre-Authorization Annotations

Controllers use annotations to declare required authorities:

```
@PreAuthorize("hasAnyAuthority('TENANT_ADMIN', 'CUSTOMER_USER')")
```

| Annotation | Authorities |
|------------|-------------|
| `hasAuthority('SYS_ADMIN')` | System admin only |
| `hasAuthority('TENANT_ADMIN')` | Tenant admin only |
| `hasAnyAuthority('TENANT_ADMIN', 'CUSTOMER_USER')` | Tenant admin or customer |
| `hasAnyAuthority('SYS_ADMIN', 'TENANT_ADMIN', 'CUSTOMER_USER')` | Any authenticated user |

## Rate Limiting

The platform implements multi-level rate limiting via `RateLimitProcessingFilter`:

| Limit Type | Scope | Description |
|------------|-------|-------------|
| REST_REQUESTS_PER_TENANT | Tenant | API calls per tenant |
| REST_REQUESTS_PER_CUSTOMER | Customer | API calls per customer |
| Telemetry messages | Device | Messages per second |
| Entity creates | Tenant | Entities per time window |

### HAProxy Rate Limiting (Microservices)

| Limit | Value | Description |
|-------|-------|-------------|
| conn_rate | 100 req/10s | Connection rate per source |
| http_req_rate | 300 req/1min | HTTP request rate |
| max connections | 50 per server | Backend connection limit |

**Rate Limit Response:**
```json
{
  "status": 429,
  "message": "Too many requests",
  "errorCode": 33
}
```

System administrators are exempt from rate limits.

## Async Operations

Some operations return `DeferredResult` for async processing:

```mermaid
sequenceDiagram
    participant C as Client
    participant API as REST API
    participant SVC as Async Service

    C->>API: GET /api/plugins/telemetry/...
    API->>SVC: Start async operation
    API-->>C: HTTP connection held

    SVC->>SVC: Process data
    SVC-->>API: Result ready

    API-->>C: 200 OK + Data
```

## Bulk Operations and Import/Export

### Bulk Import Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/device/bulk_import` | POST | Import devices from CSV |
| `/api/asset/bulk_import` | POST | Import assets from CSV |
| `/api/edge/bulk_import` | POST | Import edges from CSV |

**Bulk Import Request:**
```json
{
  "file": "base64-encoded-csv-content",
  "mapping": {
    "columns": [
      {"type": "NAME", "index": 0},
      {"type": "TYPE", "index": 1},
      {"type": "LABEL", "index": 2},
      {"type": "SHARED_ATTRIBUTE", "key": "location", "index": 3}
    ]
  }
}
```

**Bulk Import Response:**
```json
{
  "created": 50,
  "updated": 10,
  "errors": 2,
  "errorsList": [
    {"line": 5, "message": "Invalid device name"}
  ]
}
```

### Export/Import Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/ruleChains/export` | GET | Export rule chains as JSON |
| `/api/ruleChains/import` | POST | Import rule chains from JSON |
| `/api/images/{type}/{key}/export` | GET | Export image data |
| `/api/image/import` | PUT | Import image (multipart) |

**Rule Chain Export Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| limit | Integer | Max rule chains to export |

**Rule Chain Import Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| overwrite | Boolean | Overwrite existing rule chains |

### Version Control Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/entities/vc/create` | POST | Save entity versions to git |
| `/api/entities/vc/load` | POST | Load entity versions from git |
| `/api/entities/vc/compare` | POST | Compare entity versions |

## OpenAPI/Swagger

The platform exposes OpenAPI documentation:

| URL | Description |
|-----|-------------|
| `/swagger-ui.html` | Swagger UI |
| `/v3/api-docs` | OpenAPI 3.0 JSON |

### API Documentation Annotations

Controllers use annotations for documentation:

```
@ApiOperation(
    value = "Get Device (getDeviceById)",
    notes = "Fetch the Device object based on the provided Device Id..."
)
@Parameter(description = "Device ID", required = true)
```

## Best Practices

### Client Implementation

1. **Token Management**
   - Store tokens securely
   - Refresh before expiration
   - Handle 401 responses by re-authenticating

2. **Error Handling**
   - Check HTTP status codes
   - Parse error response for details
   - Implement retry logic for transient errors

3. **Pagination**
   - Use reasonable page sizes (20-100)
   - Iterate through all pages for full datasets
   - Use `hasNext` to detect end of data

4. **Rate Limiting**
   - Implement backoff on 429 responses
   - Batch operations when possible
   - Cache responses where appropriate

### Performance Tips

1. **Use Specific Endpoints**
   - Use `/api/device/{id}` instead of listing all
   - Request only needed telemetry keys
   - Use appropriate time ranges

2. **Batch Operations**
   - Use bulk import endpoints
   - Batch telemetry saves
   - Use entity queries for complex searches

3. **Caching**
   - Cache device profiles and configurations
   - Cache user permissions
   - Use ETags where supported

## Implementation Details

### BaseController Architecture

**BaseController** is the abstract base class for all REST controllers:
- Marked with `@TbCoreComponent` annotation
- Autowired with 50+ service dependencies
- Provides common validation and permission utilities
- Centralized exception handling via `@ExceptionHandler`

### Error Handling Architecture

**ThingsboardErrorResponseHandler** (`@RestControllerAdvice`) maps exceptions:

| Exception Type | HTTP Status | Error Code |
|----------------|-------------|------------|
| `ThingsboardException(GENERAL)` | 500 | 2 |
| `ThingsboardException(AUTHENTICATION)` | 401 | 10 |
| `ThingsboardException(JWT_TOKEN_EXPIRED)` | 401 | 11 |
| `ThingsboardException(PERMISSION_DENIED)` | 403 | 20 |
| `ThingsboardException(ITEM_NOT_FOUND)` | 404 | 32 |
| `TbRateLimitsException` | 429 | 33 |
| `DataAccessException` | 500 | 46 |
| `MaxPayloadSizeExceededException` | 413 | - |

### Permission Check Flow

Three-step verification pattern in controllers:
```
1. Parse ID from request: EdgeId edgeId = new EdgeId(toUUID(strEdgeId))
2. Check permission: checkEdgeId(edgeId, Operation.READ)
   - Validates ID format
   - Loads entity from database with tenant context
   - Calls accessControlService.checkPermission()
3. Execute service call with verified entity
```

**DefaultAccessControlService** enforces authority-based permissions:
- `SYS_ADMIN`, `TENANT_ADMIN`, `CUSTOMER_USER` authorities
- Throws `ThingsboardException(PERMISSION_DENIED)` on failure

### Rate Limiting Implementation

**RateLimitProcessingFilter** (OncePerRequestFilter):
- Checks `LimitedApi.REST_REQUESTS_PER_TENANT` first
- Checks `LimitedApi.REST_REQUESTS_PER_CUSTOMER` for customer users
- System admins exempt from rate limits
- Returns 429 with `TbRateLimitsException` on limit exceeded

### Request Validation

**Validator** utility provides:
- `validateEntityId()` - entity ID format validation
- `validateString()` - non-empty string validation
- `validatePositiveNumber()` - numeric bounds checking
- `isValidProperty()` - regex validation (`^[\\p{L}0-9_-]+$`) for sort properties

### Pagination Implementation

**PageLink** creation with validation:
- `pageSize` bounded (max 1000)
- `sortProperty` validated against allowed characters
- `SortOrder.Direction` parsed from string (ASC/DESC)

**TimePageLink** extends PageLink:
- Adds `startTime` and `endTime` for time-range queries
- Used by event and telemetry listing endpoints

### Async Request Handling

**DeferredResult** pattern for non-blocking I/O:
```java
DeferredResult<T> wrapFuture(ListenableFuture<T> future) {
    DeferredResult<T> result = new DeferredResult<>();
    DonAsynchron.withCallback(future, result::setResult, result::setErrorResult);
    return result;
}
```

Uses `spring.mvc.async.request-timeout` for timeout configuration.

## See Also

- [Authentication (JWT)](./authentication.md) - Detailed auth documentation
- [Device API](./device-api.md) - Device-specific endpoints
- [WebSocket Overview](./websocket-overview.md) - Real-time data
- [Subscription Model](./subscription-model.md) - Real-time subscriptions
