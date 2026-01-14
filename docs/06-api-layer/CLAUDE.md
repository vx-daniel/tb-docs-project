# CLAUDE.md - API Layer Section

This file provides guidance for working with the API Layer documentation section.

## Section Purpose

This section documents ThingsBoard's external-facing APIs:

- **REST API**: HTTP endpoints for platform management
- **WebSocket API**: Real-time subscriptions and streaming
- **Device API**: Device-facing endpoints for telemetry/attributes
- **Authentication**: JWT tokens, API keys, OAuth2
- **RPC**: Remote procedure calls to/from devices

## File Structure

```
06-api-layer/
├── README.md              # API layer overview and architecture
├── rest-api-overview.md   # REST controller organization, pagination
├── authentication.md      # JWT, API keys, OAuth2 flows
├── device-api.md          # Device endpoints (telemetry, attributes)
├── rpc-api.md             # Client-side and server-side RPC
├── alarm-api.md           # Alarm management and propagation
├── websocket-overview.md  # Real-time subscription architecture
├── subscription-model.md  # Entity data subscriptions
└── notifications.md       # Notification center and delivery
```

## Writing Guidelines

### Audience

Application developers integrating with ThingsBoard APIs. Assume familiarity with REST concepts and HTTP but not ThingsBoard-specific patterns.

### Content Pattern

API documents should include:

1. **Overview** - What the API does and common use cases
2. **Authentication** - How to authenticate requests
3. **Endpoints** - URL patterns, methods, parameters
4. **Request/Response** - Payload structures with examples
5. **Error Handling** - Error codes and resolution
6. **Rate Limits** - Throttling information
7. **Examples** - Working curl/code examples
8. **Pitfalls** - Common mistakes

### Endpoint Documentation Pattern

For each API endpoint:

```markdown
### Endpoint Name

**Method**: `GET` / `POST` / `PUT` / `DELETE`

**URL**: `/api/v1/resource/{id}`

**Authentication**: JWT Token / API Key / Device Token

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | UUID | Yes | Resource identifier |

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| pageSize | int | 10 | Results per page |

#### Request Body

\`\`\`json
{
  "field": "value"
}
\`\`\`

#### Response

\`\`\`json
{
  "id": "uuid",
  "field": "value"
}
\`\`\`

#### Error Codes

| Code | Description |
|------|-------------|
| 400 | Invalid request |
| 401 | Unauthorized |
| 404 | Not found |

#### Example

\`\`\`bash
curl -X GET "http://localhost:8080/api/v1/resource/123" \
  -H "X-Authorization: Bearer $JWT_TOKEN"
\`\`\`
```

### Terminology

- Use "JWT Token" not "auth token" or "bearer token"
- Use "Access Token" for device authentication
- Use "Endpoint" not "route" or "URL"
- Use "Request Body" not "payload" for HTTP body
- Use "Subscription" not "listener" for WebSocket patterns

### Diagrams

Use Mermaid diagrams to show:

- Authentication flow (`sequenceDiagram`)
- API architecture (`graph TB`)
- Subscription lifecycle (`sequenceDiagram`)
- Error handling flow (`flowchart`)

### Technology-Agnostic Rule

Focus on API contracts, not implementation:

**DO**: "The login endpoint returns a JWT token valid for 15 minutes"
**DON'T**: "JwtTokenFactory creates JwtToken using io.jsonwebtoken.Jwts builder"

**DO**: "WebSocket subscriptions use a command-based protocol with JSON messages"
**DON'T**: "TelemetryWebSocketHandler extends TextWebSocketHandler with TelemetrySubscriptionService"

**DO**: "Pagination uses pageSize and page parameters with a hasNext indicator"
**DON'T**: "PageDataIterableByTenantIdEntityId implements Iterable<PageData<T>>"

## Reference Sources

When updating this section, cross-reference:

- `ref/thingsboard.github.io-master/docs/api/` - Official API documentation
- `ref/thingsboard-master/application/src/main/java/org/thingsboard/server/controller/` - REST controllers
- `ref/thingsboard-master/application/src/main/java/org/thingsboard/server/service/ws/` - WebSocket handlers
- `ref/thingsboard-master/dao/src/main/java/org/thingsboard/server/dao/model/` - Data models

## Related Sections

- `05-transport-layer/` - Device connectivity (transport-level)
- `09-security/` - Authentication and authorization details
- `10-frontend/` - UI consuming these APIs
- `02-core-concepts/` - Entity structures returned by APIs

## Common Tasks

### Documenting a New REST Endpoint

1. Identify the controller and method
2. Document URL, method, and authentication
3. List all parameters (path, query, body)
4. Show request and response JSON examples
5. Document all possible error codes
6. Add working curl example
7. Update REST API overview if new controller

### Documenting WebSocket Commands

1. Document the command name and direction
2. Show the JSON message structure
3. Document subscription parameters
4. Show example subscription flow
5. Document unsubscription

### Adding Authentication Examples

1. Show complete flow from login to API call
2. Include token refresh examples
3. Document header format exactly
4. Show error responses for auth failures

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/06-api-layer/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **api-documenter** | `/api-documenter` | REST API documentation, OpenAPI specs, examples |
| **websocket-engineer** | `/websocket-engineer` | WebSocket subscriptions, real-time patterns |
| **spring-boot-expert** | `/spring-boot-expert` | REST controller patterns, Spring Security |
| **technical-writer** | `/technical-writer` | Clear API documentation structure |

### When to Use Each Skill

- **Documenting REST endpoints**: Use `/api-documenter` for structure and examples
- **Explaining WebSocket subscriptions**: Use `/websocket-engineer` for real-time patterns
- **Documenting authentication**: Use `/spring-boot-expert` for Spring Security patterns
- **Writing API guides**: Use `/technical-writer` for developer-friendly content

## Key API Concepts

When documenting APIs, emphasize:

| Concept | Key Points |
|---------|------------|
| **JWT Authentication** | Login returns access + refresh tokens |
| **Token Refresh** | Refresh token to get new access token |
| **Pagination** | pageSize, page, hasNext pattern |
| **Entity ID Format** | `{entityType}:{uuid}` format |
| **WebSocket Commands** | JSON command-based protocol |
| **Subscription Scope** | Entity, time-series, alarms |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Missing Bearer prefix | JWT must be sent as `Bearer $TOKEN` |
| Token expiration | Access tokens expire, use refresh |
| Wrong Content-Type | JSON endpoints need `application/json` |
| Case-sensitive IDs | Entity IDs are case-sensitive UUIDs |
| WebSocket reconnection | Subscriptions lost on disconnect |
| Pagination off-by-one | Page numbers start at 0 |

## Authentication Documentation

For authentication docs, ensure coverage of:

| Flow | Content |
|------|---------|
| **Login** | Username/password → JWT tokens |
| **Refresh** | Refresh token → new access token |
| **API Key** | Header-based authentication |
| **Device Token** | Access token in device requests |
| **OAuth2** | External provider integration |

## WebSocket Documentation

For WebSocket docs, ensure coverage of:

| Aspect | Content |
|--------|---------|
| **Connection** | URL, authentication header |
| **Commands** | Subscribe, unsubscribe, ping |
| **Subscriptions** | Telemetry, attributes, alarms |
| **Message Format** | JSON structure for each type |
| **Reconnection** | Re-establishing subscriptions |
| **Error Handling** | Error message format |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
