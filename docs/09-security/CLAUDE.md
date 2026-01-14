# CLAUDE.md - Security Section

This file provides guidance for working with the Security documentation section.

## Section Purpose

This section documents ThingsBoard's security infrastructure:

- **Authentication**: JWT tokens, OAuth2, 2FA, device credentials
- **Authorization**: Role-based access control (RBAC), user authorities
- **Tenant Isolation**: Multi-layer boundary enforcement
- **Rate Limiting**: API throttling, usage quotas

## File Structure

```
09-security/
├── README.md              # Security overview and architecture
├── authentication.md      # JWT, OAuth2, 2FA, session management
├── authorization.md       # RBAC, authorities, permission model
├── tenant-isolation.md    # Defense-in-depth isolation layers
└── rate-limiting.md       # API limits, rate controls, quotas
```

## Writing Guidelines

### Audience

Security engineers and platform administrators implementing secure deployments. Assume familiarity with security concepts but not necessarily with ThingsBoard's specific security patterns.

### Content Pattern

Security documents should include:

1. **Overview** - What the security mechanism does
2. **Threat Model** - What attacks it protects against
3. **Configuration** - Settings with examples
4. **Integration** - How it works with other components
5. **Best Practices** - Recommended configurations
6. **Pitfalls** - Security mistakes to avoid
7. **See Also** - Related documentation

### Security Documentation Pattern

For security concepts:

```markdown
## Security Feature

**Purpose**: What it protects

### Threat Model

| Threat | Protection |
|--------|------------|
| ... | ... |

### Configuration

\`\`\`yaml
# Security settings
\`\`\`

### Best Practices

1. Recommendation with rationale
2. Configuration guidance
3. Monitoring suggestions

### Common Pitfalls

| Pitfall | Risk | Mitigation |
|---------|------|------------|
| ... | ... | ... |
```

### Terminology

- Use "JWT Token" for JSON Web Tokens
- Use "Access Token" for device authentication tokens
- Use "Refresh Token" for session renewal tokens
- Use "Authority" for user permission levels (SYS_ADMIN, TENANT_ADMIN, CUSTOMER_USER)
- Use "Tenant Isolation" for multi-tenant boundary enforcement
- Use "Rate Limit" for request throttling

### Diagrams

Use Mermaid diagrams to show:

- Authentication flow (`sequenceDiagram`)
- Authorization hierarchy (`graph TB`)
- Tenant isolation layers (`graph TB`)
- Token lifecycle (`sequenceDiagram`)

### Technology-Agnostic Rule

Focus on security concepts and configuration, not implementation:

**DO**: "JWT tokens expire after 15 minutes by default, requiring refresh token exchange"
**DON'T**: "JwtTokenFactory.setExpTime() uses System.currentTimeMillis() + 900000"

**DO**: "Tenant isolation is enforced at API, permission, actor, queue, and database layers"
**DON'T**: "TenantId.isNullUid() check in AbstractEntityService.checkTenantId()"

**DO**: "OAuth2 authentication redirects users to the configured identity provider"
**DON'T**: "OAuth2AuthorizationRequestResolver builds AuthorizationRequest from OAuth2ClientRegistration"

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/user-guide/oauth-2-support/` - OAuth2 documentation
- `~/work/viaanix/thingsboard-master/application/src/main/java/org/thingsboard/server/service/security/` - Security services
- `~/work/viaanix/thingsboard-master/common/data/src/main/java/org/thingsboard/server/common/data/security/` - Security models
- `~/work/viaanix/thingsboard-master/dao/src/main/java/org/thingsboard/server/dao/user/` - User data access

## Related Sections

- `06-api-layer/authentication.md` - API-level auth mechanisms
- `01-architecture/multi-tenancy.md` - Tenant architecture
- `02-core-concepts/entities/device.md` - Device credentials
- `05-transport-layer/` - Transport authentication

## Common Tasks

### Documenting Authentication Flow

1. Show sequence diagram of auth flow
2. Document request/response formats
3. Include token structure explanation
4. Show refresh token usage
5. Document error scenarios

### Documenting Authorization

1. Explain authority hierarchy
2. Show permission inheritance
3. Document API access by role
4. Include edge cases

### Documenting Tenant Isolation

1. Explain each isolation layer
2. Show data flow through layers
3. Document boundary enforcement
4. Include failure scenarios

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/09-security/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **security-engineer** | `/security-engineer` | Security architecture, compliance, DevSecOps |
| **jwt-expert** | `/jwt-expert` | JWT implementation, token security, validation |
| **technical-writer** | `/technical-writer` | Clear security documentation |

### When to Use Each Skill

- **Documenting authentication flows**: Use `/jwt-expert` for token handling
- **Explaining tenant isolation**: Use `/security-engineer` for defense-in-depth
- **Writing security guides**: Use `/technical-writer` for clear documentation
- **Documenting OAuth2**: Use `/security-engineer` for identity patterns

## Key Security Concepts

When documenting security, emphasize:

| Concept | Key Points |
|---------|------------|
| **JWT Structure** | Header, payload, signature; access vs refresh tokens |
| **Token Lifecycle** | Issuance, validation, refresh, expiration |
| **Authority Levels** | SYS_ADMIN > TENANT_ADMIN > CUSTOMER_USER hierarchy |
| **Tenant Boundaries** | API, permission, actor, queue, database layers |
| **Device Credentials** | Access token, X.509, MQTT basic auth, LwM2M |
| **OAuth2 Integration** | External IdP, SSO, token mapping |

## Common Pitfalls to Document

Ensure documentation covers these security issues:

| Pitfall | Description |
|---------|-------------|
| Token in URL | JWTs exposed in query strings are logged and cached |
| Long-lived tokens | Extended expiration increases attack window |
| Missing refresh | Not implementing refresh flow causes session drops |
| Weak secrets | Using default or weak JWT signing secrets |
| Authority confusion | Misunderstanding TENANT_ADMIN vs CUSTOMER_USER scope |
| Missing tenant check | Forgetting tenant ID validation in custom extensions |
| Rate limit bypass | Not applying limits to all API paths |
| OAuth2 redirect | Insecure redirect URI configuration |

## Authentication Documentation

For authentication docs, ensure coverage of:

| Flow | Content |
|------|---------|
| **Login** | Username/password → JWT pair |
| **Refresh** | Refresh token → new access token |
| **OAuth2** | External IdP → JWT mapping |
| **2FA** | Second factor verification |
| **Device Auth** | Access token / X.509 / MQTT auth |

## Authorization Documentation

For authorization docs, ensure coverage of:

| Aspect | Content |
|--------|---------|
| **Authorities** | SYS_ADMIN, TENANT_ADMIN, CUSTOMER_USER |
| **Permissions** | Entity-level access control |
| **Inheritance** | Permission flow in customer hierarchy |
| **API Access** | Which endpoints each role can access |

## Tenant Isolation Documentation

For isolation docs, ensure coverage of:

| Layer | Content |
|-------|---------|
| **API Layer** | Request validation, tenant context |
| **Permission Layer** | Entity access verification |
| **Actor Layer** | Tenant actor isolation |
| **Queue Layer** | Message routing by tenant |
| **Database Layer** | Query filtering by tenant ID |

## Rate Limiting Documentation

For rate limiting docs, ensure coverage of:

| Aspect | Content |
|--------|---------|
| **Limits** | Message count, API requests, WS connections |
| **Scopes** | Per-device, per-tenant, global |
| **Configuration** | YAML settings and defaults |
| **Monitoring** | Limit violation alerts |
| **Recovery** | Behavior after limit reset |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
