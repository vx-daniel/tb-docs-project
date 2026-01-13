# Authentication

## Overview

ThingsBoard uses JWT (JSON Web Token) based authentication for securing API access. The platform issues two types of tokens: short-lived access tokens for API calls and longer-lived refresh tokens for obtaining new access tokens. Authentication supports multiple methods including username/password, OAuth2 providers, API Keys (Personal Access Tokens), and two-factor authentication (2FA).

## Authentication Methods Summary

| Method | Use Case | Header Format |
|--------|----------|---------------|
| JWT Bearer | User sessions | `Authorization: Bearer <token>` or `X-Authorization: <token>` |
| API Key | Service integrations | `Authorization: ApiKey <key>` |
| OAuth2 | External identity providers | Redirect-based flow |
| Device Token | Device connectivity | Protocol-specific (MQTT username, HTTP path) |

## Key Behaviors

1. **Stateless Authentication**: JWTs contain all necessary user information, eliminating server-side session storage.

2. **Token Pair System**: Every successful login produces both an access token and a refresh token.

3. **Role-Based Access**: User authorities (SYS_ADMIN, TENANT_ADMIN, CUSTOMER_USER) are embedded in the token.

4. **Session Tracking**: Each token pair has a unique session ID for selective invalidation.

5. **Multi-Factor Authentication**: Optional 2FA using TOTP, SMS, email, or backup codes.

6. **OAuth2 Integration**: External identity providers can authenticate users and provision accounts.

## Authentication Flow

### Standard Login Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant F as Auth Filter
    participant P as Auth Provider
    participant U as User Service
    participant T as Token Factory

    C->>F: POST /api/auth/login<br/>{username, password}
    F->>P: Authenticate
    P->>U: Find user by email
    U-->>P: User + Credentials
    P->>P: Verify password (BCrypt)
    P->>P: Check account status

    alt Account locked
        P-->>C: 401 - Account locked
    else Password expired
        P-->>C: 401 - Credentials expired
    else 2FA required
        P->>T: Create pre-verification token
        T-->>C: Pre-verification JWT
    else Success
        P->>T: Create token pair
        T-->>C: {accessToken, refreshToken}
    end
```

### Token Refresh Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant F as Refresh Filter
    participant P as Refresh Provider
    participant O as Outdating Service
    participant T as Token Factory

    C->>F: POST /api/auth/token<br/>{refreshToken}
    F->>P: Authenticate refresh token
    P->>T: Parse refresh token claims
    T-->>P: User info from token
    P->>O: Check if token outdated

    alt Token outdated
        O-->>C: 401 - Token is outdated
    else Token valid
        P->>T: Create new token pair
        T-->>C: {accessToken, refreshToken}
    end
```

## JWT Token Structure

### Access Token Claims

| Claim | Key | Description |
|-------|-----|-------------|
| Subject | `sub` | User ID (UUID) |
| Scopes | `scopes` | List of authorities |
| User ID | `userId` | User UUID |
| First Name | `firstName` | User's first name |
| Last Name | `lastName` | User's last name |
| Enabled | `enabled` | Account enabled flag |
| Tenant ID | `tenantId` | Tenant UUID |
| Customer ID | `customerId` | Customer UUID (if applicable) |
| Is Public | `isPublic` | Public user flag |
| Session ID | `sessionId` | Unique session identifier |
| Issued At | `iat` | Token creation timestamp |
| Expiration | `exp` | Token expiration timestamp |
| Issuer | `iss` | Token issuer (configurable) |

### Token Example

```json
{
  "sub": "e4567890-e89b-12d3-a456-426614174000",
  "scopes": ["TENANT_ADMIN"],
  "userId": "e4567890-e89b-12d3-a456-426614174000",
  "firstName": "John",
  "lastName": "Doe",
  "enabled": true,
  "tenantId": "a1234567-e89b-12d3-a456-426614174000",
  "customerId": "13814000-1dd2-11b2-8080-808080808080",
  "isPublic": false,
  "sessionId": "f6789012-e89b-12d3-a456-426614174000",
  "iat": 1704067200,
  "exp": 1704076200,
  "iss": "thingsboard.io"
}
```

### Refresh Token Claims

Refresh tokens contain a subset of claims:

| Claim | Description |
|-------|-------------|
| Subject | User ID |
| Scopes | `[REFRESH_TOKEN]` |
| User ID | User UUID |
| Is Public | Public user flag |
| Session ID | Same session ID as access token |
| Issued At | Token creation timestamp |
| Expiration | Longer expiration than access token |

## User Authorities

### Authority Hierarchy

```mermaid
graph TB
    subgraph "User Roles"
        SYS[SYS_ADMIN<br/>Code: 0]
        TENANT[TENANT_ADMIN<br/>Code: 1]
        CUSTOMER[CUSTOMER_USER<br/>Code: 2]
    end

    subgraph "Special Token Types"
        REFRESH[REFRESH_TOKEN<br/>Code: 10]
        PRE[PRE_VERIFICATION_TOKEN<br/>Code: 11]
        MFA[MFA_CONFIGURATION_TOKEN<br/>Code: 12]
    end

    SYS -->|"manages"| TENANT
    TENANT -->|"manages"| CUSTOMER
```

### Authority Descriptions

| Authority | Code | Description | Scope |
|-----------|------|-------------|-------|
| SYS_ADMIN | 0 | System administrator | Full platform access |
| TENANT_ADMIN | 1 | Tenant administrator | Tenant-level access |
| CUSTOMER_USER | 2 | Customer user | Limited to assigned devices |
| REFRESH_TOKEN | 10 | Refresh token scope | Token renewal only |
| PRE_VERIFICATION_TOKEN | 11 | Pre-2FA verification | Limited until 2FA complete |
| MFA_CONFIGURATION_TOKEN | 12 | 2FA setup | Configure 2FA settings |

## JWT Configuration

### Settings Model

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| tokenExpirationTime | Integer | 9000 | Access token TTL (seconds) |
| refreshTokenExpTime | Integer | 604800 | Refresh token TTL (seconds) |
| tokenIssuer | String | "thingsboard.io" | JWT issuer claim |
| tokenSigningKey | String | (generated) | Base64-encoded signing key |

### Token Signing

- **Algorithm**: HS512 (HMAC with SHA-512)
- **Key Length**: Minimum 512 bits (64 bytes)
- **Key Storage**: Stored securely in system settings

### Configuration Example

```json
{
  "tokenExpirationTime": 9000,
  "refreshTokenExpTime": 604800,
  "tokenIssuer": "thingsboard.io",
  "tokenSigningKey": "dkVTUzU2M2VMWUNwVVltTUhQU2o5SUM0..."
}
```

## Two-Factor Authentication (2FA)

### Supported Providers

```mermaid
graph LR
    subgraph "2FA Methods"
        TOTP[TOTP<br/>Authenticator App]
        SMS[SMS<br/>Text Message]
        EMAIL[Email<br/>Code via Email]
        BACKUP[Backup Codes<br/>One-time Use]
    end

    USER[User] --> TOTP
    USER --> SMS
    USER --> EMAIL
    USER --> BACKUP
```

| Provider | Description | Use Case |
|----------|-------------|----------|
| TOTP | Time-based One-Time Password | Authenticator apps (Google, Authy) |
| SMS | SMS verification code | Mobile phone verification |
| EMAIL | Email verification code | Email-based verification |
| BACKUP_CODE | Pre-generated codes | Recovery when primary unavailable |

### 2FA Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Endpoint
    participant M as 2FA Service
    participant T as Token Factory

    C->>A: Login with credentials
    A->>A: Verify password
    A->>M: Check if 2FA enabled
    M-->>A: 2FA required
    A->>T: Create pre-verification token
    T-->>C: Pre-verification token

    C->>A: Submit 2FA code<br/>+ pre-verification token
    A->>M: Verify code

    alt Code valid
        M-->>A: Verified
        A->>T: Create full token pair
        T-->>C: {accessToken, refreshToken}
    else Code invalid
        M-->>C: 401 - Invalid code
    end
```

### 2FA Service Interface

```
isTwoFaEnabled(tenantId, user) → boolean
isEnforceTwoFaEnabled(tenantId, user) → boolean
checkProvider(tenantId, providerType) → void
prepareVerificationCode(user, providerType, checkLimits) → void
checkVerificationCode(user, providerType, code, checkLimits) → boolean
generateNewAccountConfig(user, providerType) → TwoFaAccountConfig
```

## Password Policy

### Policy Configuration

```mermaid
graph TB
    subgraph "Password Requirements"
        LEN[Length<br/>Min: 8, Max: 72]
        UPPER[Uppercase<br/>Min: 1]
        LOWER[Lowercase<br/>Min: 1]
        DIGIT[Digits<br/>Min: 1]
        SPECIAL[Special Chars<br/>Min: 1]
    end

    subgraph "Expiration Rules"
        EXP[Expiration<br/>Days until change]
        REUSE[Reuse Frequency<br/>Days before reuse]
    end

    PASSWORD[Password] --> LEN
    PASSWORD --> UPPER
    PASSWORD --> LOWER
    PASSWORD --> DIGIT
    PASSWORD --> SPECIAL
    PASSWORD --> EXP
    PASSWORD --> REUSE
```

### Policy Fields

| Field | Type | Description |
|-------|------|-------------|
| minimumLength | Integer | Minimum password length |
| maximumLength | Integer | Maximum password length |
| minimumUppercaseLetters | Integer | Required uppercase count |
| minimumLowercaseLetters | Integer | Required lowercase count |
| minimumDigits | Integer | Required digit count |
| minimumSpecialCharacters | Integer | Required special char count |
| allowWhitespaces | Boolean | Allow spaces in password |
| forceUserToResetPasswordIfNotValid | Boolean | Force reset on policy violation |
| passwordExpirationPeriodDays | Integer | Days until password expires |
| passwordReuseFrequencyDays | Integer | Days before password reuse |

### Example Policy

```json
{
  "minimumLength": 8,
  "maximumLength": 72,
  "minimumUppercaseLetters": 1,
  "minimumLowercaseLetters": 1,
  "minimumDigits": 1,
  "minimumSpecialCharacters": 1,
  "allowWhitespaces": false,
  "forceUserToResetPasswordIfNotValid": false,
  "passwordExpirationPeriodDays": 90,
  "passwordReuseFrequencyDays": 30
}
```

## Security Settings

### Global Settings

| Field | Type | Description |
|-------|------|-------------|
| passwordPolicy | UserPasswordPolicy | Password requirements |
| maxFailedLoginAttempts | Integer | Attempts before lockout |
| userLockoutNotificationEmail | String | Admin notification email |
| mobileSecretKeyLength | Integer | Mobile secret key length |
| userActivationTokenTtl | Integer | Activation link TTL (hours, 1-24) |
| passwordResetTokenTtl | Integer | Reset link TTL (hours, 1-24) |

### Account Lockout

```mermaid
stateDiagram-v2
    [*] --> Active
    Active --> Warning: Failed attempt
    Warning --> Warning: More failures
    Warning --> Active: Successful login
    Warning --> Locked: Max attempts reached
    Locked --> Active: Admin unlock
    Locked --> Active: Timeout expires
```

## Token Invalidation

### Outdating Service

The platform maintains a cache of invalidated sessions and user credentials. Tokens are checked against this cache on each request.

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant O as Outdating Service
    participant CACHE as Cache

    C->>A: Request with JWT
    A->>O: Is token outdated?
    O->>CACHE: Check user ID
    CACHE-->>O: Invalidation timestamp
    O->>O: Compare issue time vs invalidation

    alt Issued before invalidation
        O-->>A: Token outdated
        A-->>C: 401 Unauthorized
    else Issued after invalidation
        O-->>A: Token valid
        A-->>C: Process request
    end
```

### Invalidation Events

| Event | Trigger | Effect |
|-------|---------|--------|
| UserCredentialsInvalidationEvent | Password change | All user tokens invalidated |
| UserSessionInvalidationEvent | Logout | Specific session invalidated |
| UserAuthDataChangedEvent | Security change | User tokens invalidated |

## OAuth2 Authentication

### Supported Providers

| Provider | Mapper Type | Description |
|----------|-------------|-------------|
| Basic | BASIC | Standard OAuth2 providers |
| GitHub | GITHUB | GitHub-specific mapping |
| Apple | APPLE | Apple Sign-In |
| Custom | CUSTOM | Custom attribute mapping |

### OAuth2 Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Platform
    participant O as OAuth Provider
    participant M as Client Mapper

    C->>P: Initiate OAuth2 login
    P->>O: Redirect to provider
    O->>O: User authenticates
    O->>P: Authorization code callback
    P->>O: Exchange code for token
    O-->>P: OAuth access token
    P->>M: Map user attributes
    M->>M: Find or create user
    M-->>P: SecurityUser
    P->>P: Create JWT token pair
    P-->>C: Redirect with tokens
```

### OAuth2 Success Response

After successful OAuth2 authentication, the user is redirected with tokens in URL parameters:

```
https://app.example.com/?accessToken=eyJ...&refreshToken=eyJ...
```

## API Endpoints

### Authentication Endpoints

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/auth/login` | Username/password login | No |
| POST | `/api/auth/token` | Refresh access token | Refresh token |
| GET | `/api/auth/user` | Get current user info | Yes |
| POST | `/api/auth/logout` | Record logout (audit) | Yes |
| POST | `/api/auth/changePassword` | Change password | Yes |

### No-Auth Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/noauth/userPasswordPolicy` | Get password policy |
| GET | `/api/noauth/activate` | Check activation token |
| POST | `/api/noauth/activate` | Activate user account |
| POST | `/api/noauth/resetPasswordByEmail` | Request password reset |
| GET | `/api/noauth/resetPassword` | Check reset token |
| POST | `/api/noauth/resetPassword` | Reset password |

### Login Request

```
POST /api/auth/login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "securePassword123!"
}
```

### Login Response

```json
{
  "token": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJlNDU2...",
  "refreshToken": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJl..."
}
```

### Token Refresh Request

```
POST /api/auth/token
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJl..."
}
```

### Change Password Request

```
POST /api/auth/changePassword
Content-Type: application/json
X-Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...

{
  "currentPassword": "oldPassword123!",
  "newPassword": "newSecurePassword456!"
}
```

## Using Tokens

### Request Authentication

Include the access token in the `X-Authorization` header:

```
GET /api/devices
X-Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

### Token Expiration Handling

```mermaid
graph TB
    REQ[API Request] --> CHECK{Token<br/>expired?}
    CHECK -->|No| PROCESS[Process Request]
    CHECK -->|Yes| REFRESH{Have refresh<br/>token?}
    REFRESH -->|Yes| RENEW[Call /api/auth/token]
    REFRESH -->|No| LOGIN[Redirect to login]
    RENEW --> RETRY[Retry original request]
```

### Error Responses

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 401 | AUTHENTICATION | Invalid credentials |
| 401 | JWT_TOKEN_EXPIRED | Access token expired |
| 401 | CREDENTIALS_EXPIRED | Password expired |
| 403 | PERMISSION_DENIED | Insufficient privileges |
| 429 | TOO_MANY_REQUESTS | Rate limit exceeded |

### Error Response Example

```json
{
  "status": 401,
  "message": "JWT token expired",
  "errorCode": 11,
  "timestamp": "2024-01-09T12:00:00.000+00:00"
}
```

## Public User Authentication

### Public Customer Access

Devices can authenticate using public customer credentials for dashboard access:

```mermaid
sequenceDiagram
    participant D as Dashboard
    participant P as Platform

    D->>P: POST /api/auth/login<br/>{publicId: "public-customer-uuid"}
    P->>P: Validate public customer
    P->>P: Create limited token
    P-->>D: Token with CUSTOMER_USER scope

    D->>P: GET /api/dashboards/{id}
    Note over P: Access limited to<br/>public customer's entities
```

## API Key Authentication (Personal Access Tokens)

API Keys provide long-lived authentication for service integrations without requiring username/password login.

### API Key Structure

```json
{
  "id": {
    "entityType": "API_KEY",
    "id": "apikey-uuid"
  },
  "tenantId": { "id": "tenant-uuid" },
  "userId": { "id": "user-uuid" },
  "name": "Integration Service Key",
  "value": "AbCdEf123456...",
  "expirationTime": 1735689600000,
  "enabled": true
}
```

### API Key Usage

```
GET /api/devices
Authorization: ApiKey AbCdEf123456...
```

### API Key Features

| Feature | Description |
|---------|-------------|
| Expiration | Optional expiration timestamp |
| Enable/Disable | Can be disabled without deletion |
| User Association | Inherits permissions from associated user |
| XSS Protection | Values sanitized against injection |

### API Key Processing

```mermaid
sequenceDiagram
    participant C as Client
    participant F as ApiKey Filter
    participant P as ApiKey Provider
    participant DB as Database

    C->>F: Request with ApiKey header
    F->>F: Extract key from header
    F->>P: Authenticate key
    P->>DB: Find ApiKey by value
    DB-->>P: ApiKey entity
    P->>P: Check enabled status
    P->>P: Check expiration time
    alt Valid key
        P-->>F: SecurityUser
        F-->>C: Process request
    else Invalid/expired
        P-->>C: 401 Unauthorized
    end
```

## Device Credentials

Devices authenticate using credentials configured in their device profile.

### Credential Types

| Type | Description | Use Case |
|------|-------------|----------|
| ACCESS_TOKEN | Simple token authentication | Basic device connectivity |
| X509_CERTIFICATE | X.509 certificate authentication | Secure TLS-based auth |
| MQTT_BASIC | Username/password for MQTT | MQTT broker authentication |
| LWM2M_CREDENTIALS | LwM2M security configuration | Lightweight M2M devices |

### Access Token Authentication

```mermaid
sequenceDiagram
    participant D as Device
    participant T as Transport
    participant A as Auth Service

    D->>T: Connect with access token
    T->>A: Validate token
    A->>A: Lookup device by token
    alt Token valid
        A-->>T: DeviceAuthResult (success)
        T-->>D: Connection established
    else Token invalid
        A-->>T: DeviceAuthResult (failure)
        T-->>D: Connection rejected
    end
```

### X.509 Certificate Authentication

Supports both JKS keystore and PEM certificate formats for mutual TLS authentication.

```mermaid
graph LR
    D[Device] -->|TLS + Client Cert| T[Transport]
    T -->|Extract CN/Fingerprint| A[Auth Service]
    A -->|Match Credentials| DB[(Database)]
```

## Security Filter Chain

The platform processes requests through a series of security filters:

```mermaid
graph TB
    REQ[Incoming Request] --> AEH[Auth Exception Handler]
    AEH --> RLPF[Rest Login Filter<br/>/api/auth/login]
    RLPF --> RPLF[Rest Public Login Filter<br/>/api/auth/login/public]
    RPLF --> JTAF[JWT Token Auth Filter]
    JTAF --> AKTF[API Key Token Auth Filter]
    AKTF --> RTPF[Refresh Token Filter<br/>/api/auth/token]
    RTPF --> RLF[Rate Limit Filter]
    RLF --> CTRL[Controller]
```

### Filter Responsibilities

| Filter | Path | Function |
|--------|------|----------|
| RestLoginProcessingFilter | `/api/auth/login` | Username/password authentication |
| RestPublicLoginProcessingFilter | `/api/auth/login/public` | Public customer login |
| JwtTokenAuthenticationProcessingFilter | `/**` | JWT token validation |
| ApiKeyTokenAuthenticationProcessingFilter | `/**` | API key validation |
| RefreshTokenProcessingFilter | `/api/auth/token` | Token refresh |
| RateLimitProcessingFilter | `/**` | Rate limiting enforcement |

## Rate Limiting

Authentication endpoints have rate limiting to prevent brute force attacks:

| Endpoint | Default Limit |
|----------|---------------|
| Login | Per IP and account |
| Password Reset | 5 per hour per user |
| Token Refresh | Per user |
| REST Requests | Per tenant/customer (configurable) |

### Rate Limit Types

| Limit Type | Scope | Description |
|------------|-------|-------------|
| REST_REQUESTS_PER_TENANT | Tenant | API calls per tenant |
| REST_REQUESTS_PER_CUSTOMER | Customer | API calls per customer |
| PASSWORD_RESET | User | Password reset attempts |
| WEBSOCKET_SESSIONS | Tenant | WebSocket connections |

System administrators are exempt from rate limits.

## Audit Logging

All authentication events are recorded in the audit log for security monitoring and compliance.

### Logged Events

| Action | Description |
|--------|-------------|
| LOGIN | Successful user login |
| LOGOUT | User logout |
| LOGIN_LOCKOUT | Account locked after failed attempts |
| CREDENTIALS_UPDATED | Password or credentials changed |
| TWO_FA_ENABLED | 2FA enabled for user |
| TWO_FA_DISABLED | 2FA disabled for user |

### Audit Log Sinks

| Sink | Description |
|------|-------------|
| Database | Default PostgreSQL storage |
| Elasticsearch | Optional for advanced search |

## Implementation Details

### JWT Token Creation

**JwtTokenFactory** handles all token generation:
- **Signing Algorithm**: HMAC-SHA512 (HS512) with 512-bit key
- **Key Management**: Base64-encoded secret stored in configuration, volatile cached with synchronized lazy initialization
- **Runtime Key Rotation**: `reload()` method allows key rotation without restart
- **Session Tracking**: Each token pair includes UUID-based session ID for granular invalidation

Token claims include: userId, scopes, firstName, lastName, enabled status, tenantId, customerId, sessionId, iat, exp, issuer.

### Token Invalidation Mechanism

**DefaultTokenOutdatingService** provides invalidation:
- **Cache Backend**: Redis-backed (`UsersSessionInvalidationRedisCache`) for distributed systems, Caffeine fallback
- **Cache Key**: Session ID or User ID
- **Validation Logic**: `token.issuedAt < cachedInvalidationTime` = token invalid
- **Event-Driven**: Uses Spring `@EventListener` on `UserAuthDataChangedEvent`

### 2FA Rate Limiting

**DefaultTwoFactorAuthService** enforces limits:
- **Code Send Limit**: `minVerificationCodeSendPeriod` (>4 seconds between sends)
- **Check Limit**: Tenant-configurable via `PlatformTwoFaSettings`
- **Failed Attempt Tracking**: `userService.increaseFailedLoginAttempts()` called on failures
- **Auto-Lock**: Account locked after `maxVerificationFailuresBeforeUserLockout` exceeded
- **Rate Limit Key**: (UserId, ProviderType) tuple for granular control

### Password Policy Implementation

**DefaultSystemSecurityService** uses `passay` library:
- Rules: Length bounds, character composition, whitespace control
- **Password History**: Stored in `userCredentials.additionalInfo` as timestamp→BCrypt hash map
- **Reuse Check**: BCrypt `matches()` for constant-time comparison
- **Expiration Check**: `createdTime + expirationDays < currentTime` triggers reset token

### OAuth2 Race Condition Handling

**AbstractOAuth2ClientMapper** prevents concurrent user creation:
- `ReentrantLock` per email during OAuth2 login
- Automatic tenant/customer creation if `allowUserCreation` enabled
- Dashboard auto-assignment by name lookup (10-item pagination limit)

### Configuration Properties

```yaml
security.jwt:
  tokenExpirationTime: 9000        # Access token TTL (seconds)
  refreshTokenExpTime: 604800      # Refresh token TTL (seconds)
  tokenIssuer: "thingsboard.io"
  tokenSigningKey: "base64-encoded-key"

security.user_login_case_sensitive: false
security.claim.allowClaimingByDefault: false
```

## Best Practices

1. **Token Storage**: Store tokens securely (HttpOnly cookies or secure storage)

2. **Token Refresh**: Implement proactive refresh before expiration

3. **Logout Handling**: Call logout endpoint to invalidate session

4. **Password Complexity**: Enforce strong password policies

5. **2FA Enforcement**: Enable 2FA for administrative accounts

6. **Token Rotation**: Rotate signing keys periodically

7. **Session Monitoring**: Monitor for suspicious session patterns

## See Also

- [REST API Overview](./rest-api-overview.md) - API structure and patterns
- [Device API](./device-api.md) - Device authentication
- [WebSocket Overview](./websocket-overview.md) - Real-time authentication
- [Security Architecture](../09-security/authentication.md) - Platform security details
