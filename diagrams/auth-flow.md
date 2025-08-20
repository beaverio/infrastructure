# Authentication Flow Architecture

## Overview
This document describes the OAuth2/OpenID Connect authentication flow using Keycloak as the identity provider in a microservices architecture with Auth Gateway pattern for session-to-JWT orchestration.

## Authentication Flow Diagrams

### OAuth2 Sign In Flow with Provider Selection

```mermaid
sequenceDiagram
    participant Client as 👤 Client
    participant AuthGW as 🔐 Auth Gateway
    participant Keycloak as 🔐 Keycloak
    participant Provider as 🌐 OAuth Provider (Google/Apple)
    participant Redis as ⚡ Session Store
    participant InternalGW as 🚪 Internal Gateway
    participant Identity as 👤 Identity Service
    participant IdentityDB as 🗄️ Identity DB

    Note over Client,IdentityDB: OAuth2 Sign In Flow with Provider Selection
    Client->>AuthGW: Access protected resource OR click login
    AuthGW->>Redis: Check for existing session
    Redis-->>AuthGW: No session found
    AuthGW->>AuthGW: Generate OAuth2 state + redirect URL
    AuthGW-->>Client: Redirect to Keycloak realm with state
    
    Client->>Keycloak: Access Keycloak login page
    Keycloak-->>Client: Show provider selection (Google, Apple, etc.)
    Client->>Keycloak: Select Google OAuth
    Keycloak-->>Client: Redirect to Google OAuth (with Keycloak client_id & redirect_uri)
    
    Client->>Provider: Login with Google credentials
    Provider->>Provider: Authenticate user
    Provider-->>Client: Redirect to Keycloak callback with provider authorization code
    Client->>Keycloak: Keycloak callback with provider authorization code
    Keycloak->>Provider: Exchange provider authorization code for user profile
    Provider-->>Keycloak: User profile data (email, name, etc.)
    Keycloak->>Keycloak: Create/update user from provider profile
    Keycloak-->>Client: Redirect to Auth Gateway callback with Keycloak authorization code
    
    Client->>AuthGW: Auth Gateway callback with Keycloak authorization code + state
    AuthGW->>AuthGW: Validate state parameter
    AuthGW->>Keycloak: Exchange authorization code for tokens
    Keycloak-->>AuthGW: Access token + ID token + Refresh token (basic claims)
    
    Note over AuthGW,IdentityDB: Get User's Workspace Context
    AuthGW->>InternalGW: Get user context using access token
    InternalGW->>Identity: GET /users/me/context (with Bearer token)
    Identity->>IdentityDB: SET SESSION 'app.workspace_id' = NULL (admin query)
    Identity->>IdentityDB: SELECT user, lastWorkspaceId, roles, workspaces FROM users WHERE keycloak_sub = ?
    IdentityDB-->>Identity: User data + workspace memberships + roles
    Identity-->>InternalGW: User profile + workspace context
    InternalGW-->>AuthGW: User context data
    
    AuthGW->>Keycloak: Token exchange for workspace-scoped JWT (custom claims)
    Keycloak-->>AuthGW: Workspace-scoped JWT with userId + workspaceId + roles
    AuthGW->>Redis: Store session (JWT + refresh token + expiry + user context)
    Note right of Redis: Session TTL: 24 hours idle, 7 days absolute
    AuthGW-->>Client: Set HTTP-only session cookie + redirect to original resource
```

### API Call with Workspace Context

```mermaid
sequenceDiagram
    participant Client as 👤 Client
    participant AuthGW as 🔐 Auth Gateway
    participant Keycloak as 🔐 Keycloak
    participant Redis as ⚡ Session Store
    participant InternalGW as 🚪 Internal Gateway
    participant Transactions as 💰 Transactions Service
    participant TransactionsDB as 🗄️ Transactions DB

    Note over Client,TransactionsDB: 1. API Call with Workspace Context
    Client->>AuthGW: GET /api/transactions/recent (session cookie)
    AuthGW->>Redis: Get session data by cookie
    Redis-->>AuthGW: Session data + workspace-scoped JWT
    AuthGW->>AuthGW: Check if JWT is expired
    
    alt JWT is valid
        AuthGW->>InternalGW: Forward request with JWT Bearer token
    else JWT is expired
        AuthGW->>Keycloak: Refresh token request
        Keycloak-->>AuthGW: New access token
        AuthGW->>Redis: Update session with new JWT
        AuthGW->>InternalGW: Forward request with new JWT Bearer token
    end
    
    InternalGW->>Transactions: Route request with JWT
    Transactions->>Transactions: Extract workspaceId from JWT claims
    Transactions->>TransactionsDB: SET SESSION 'app.workspace_id' = 'workspace-123'
    Transactions->>TransactionsDB: SELECT * FROM transactions ORDER BY created_at DESC LIMIT 10
    Note right of TransactionsDB: RLS: WHERE workspace_id = current_setting('app.workspace_id')
    TransactionsDB-->>Transactions: Workspace-filtered results
    Transactions-->>InternalGW: Transaction data
    InternalGW-->>AuthGW: Pass through response
    AuthGW-->>Client: JSON response

    Note over Client,TransactionsDB: 2. Session Expiry Handling
    Client->>AuthGW: API call after session expires
    AuthGW->>Redis: Check session
    Redis-->>AuthGW: Session expired or not found
    AuthGW-->>Client: 401 Unauthorized + redirect to login
```

## Security Standards Alignment

### ✅ OAuth2/OpenID Connect Compliance
- **Authorization Code Flow**: Industry standard for web applications
- **PKCE Support**: Code challenge/verifier for enhanced security
- **JWT Tokens**: Stateless, cryptographically signed tokens
- **Refresh Token Rotation**: Enhanced security with token rotation
- **Scope-based Authorization**: Fine-grained permission control

### ✅ Zero Trust Architecture
- **No Direct Service Access**: All services private except Auth Gateway
- **Token Validation at Internal Gateway**: Central authentication checkpoint
- **Service-to-Service Authentication**: Internal JWT validation
- **Session Management**: Server-side session storage in Redis

### ✅ Security Headers & CORS
```http
# Security Headers (Auth Gateway should implement)
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'

# CORS Configuration
Access-Control-Allow-Origin: https://yourapp.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
```

## Token Validation Strategy

### At Internal Gateway Level
```yaml
# Internal Gateway JWT Validation Configuration
security:
  oauth2:
    resourceserver:
      jwt:
        issuer-uri: http://keycloak:8090/realms/dev
        jwk-set-uri: http://keycloak:8090/realms/dev/protocol/openid-connect/certs
```

### Token Structure
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "keycloak-key-id"
  },
  "payload": {
    "exp": 1723123456,
    "iat": 1723120456,
    "iss": "http://keycloak:8090/realms/dev",
    "sub": "user-uuid",
    "preferred_username": "john.doe",
    "email": "john@example.com",
    "workspaceId": "workspace-123",
    "roles": ["user", "manager"],
    "scope": "openid profile email"
  }
}
```

## Session Management & Token Lifecycle

### Redis Session Structure
```json
{
  "sessionId": "session-uuid-12345",
  "userId": "keycloak-user-sub-id", 
  "keycloakSub": "google-oauth-sub-id",
  "workspaceId": "workspace-123",
  "workspaceName": "ACME Corp",
  "roles": ["user", "manager"],
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIs...",
  "accessTokenExpiry": "2024-08-19T15:30:00Z",
  "refreshTokenExpiry": "2024-08-26T14:30:00Z",
  "createdAt": "2024-08-19T14:30:00Z",
  "lastAccessedAt": "2024-08-19T14:45:00Z"
}
```

### Session Expiry Strategy
- **Idle Timeout**: 24 hours (session deleted if no activity)
- **Absolute Timeout**: 7 days (session deleted regardless of activity)
- **Access Token TTL**: 15 minutes (short-lived for security)
- **Refresh Token TTL**: 7 days (matches absolute session timeout)

### Token Refresh Logic
When a request comes in with a session cookie:

1. **Check Session Exists**: Auth Gateway looks up session in Redis
2. **Check Access Token Expiry**: Compare `accessTokenExpiry` with current time
3. **If Expired**: Use `refreshToken` to get new access token from Keycloak
4. **Update Session**: Store new access token and expiry in Redis
5. **Update Last Accessed**: Reset idle timeout
6. **Forward Request**: Send request to Internal Gateway with valid JWT

### Session Cleanup
- **Redis TTL**: Sessions automatically expire based on idle/absolute timeouts
- **Logout**: Explicit session deletion from Redis + Keycloak logout
- **Token Revocation**: Refresh tokens revoked on logout for security

## Role-Based Access Control (RBAC)

### Keycloak Configuration
```json
{
  "realm": "dev",
  "clients": [{
    "clientId": "beaver-auth-gateway",
    "clientAuthenticatorType": "client-secret",
    "redirectUris": ["http://localhost:8080/callback"],
    "webOrigins": ["http://localhost:3000"],
    "standardFlowEnabled": true,
    "implicitFlowEnabled": false,
    "directAccessGrantsEnabled": false
  }],
  "roles": {
    "realm": ["admin", "user", "manager"],
    "client": {
      "beaver-auth-gateway": ["transactions:read", "transactions:write", "users:read"]
    }
  }
}
```

### Authorization Matrix
| Role | Identity Service | Transactions Service | Internal Gateway Access |
|------|-----------------|---------------------|----------------|
| **user** | Read own profile | Read own transactions | Basic routes |
| **manager** | Read team profiles | Read team transactions | Manager routes |
| **admin** | Full access | Full access | All routes |

## API Request Flow Details

### Subsequent API Calls (After Login)
Every API call after login follows this pattern:

```
1. Client → Auth Gateway (with session cookie)
2. Auth Gateway → Redis (get session by cookie)
3. Redis → Auth Gateway (session data + tokens)
4. Auth Gateway checks access token expiry:
   - If valid: Forward to Internal Gateway
   - If expired: Refresh token with Keycloak, update Redis, then forward
5. Auth Gateway → Internal Gateway (with JWT Bearer token)
6. Internal Gateway validates JWT and routes to service
7. Service extracts workspace context and executes with RLS
8. Response flows back unchanged
```

### What Auth Gateway Stores vs Forwards
**Stored in Redis Session:**
- Access token (for forwarding to services)
- Refresh token (for getting new access tokens)
- User workspace context (for quick lookups)
- Session metadata (expiry, last accessed, etc.)

**Forwarded to Services:**
- JWT access token in `Authorization: Bearer` header
- No cookies or session data passed to internal services

### Error Scenarios
**Session Not Found:**
```http
HTTP/1.1 401 Unauthorized
{
  "error": "session_not_found",
  "message": "Please log in",
  "loginUrl": "/auth/login"
}
```

**Token Refresh Failed:**
```http
HTTP/1.1 401 Unauthorized  
{
  "error": "refresh_failed",
  "message": "Session expired, please log in again",
  "loginUrl": "/auth/login"
}
```

**Access Token Expired (transparent to client):**
- Auth Gateway automatically refreshes token
- Client receives normal response
- Session `lastAccessedAt` updated to extend idle timeout

## Development Configuration

### Local Environment Variables
```env
# Keycloak Configuration
KEYCLOAK_REALM=dev
KEYCLOAK_CLIENT_ID=beaver-auth-gateway
KEYCLOAK_CLIENT_SECRET=your-client-secret
KEYCLOAK_ISSUER_URL=http://keycloak:8090/realms/dev

# Session Configuration  
REDIS_HOST=auth-redis
REDIS_PORT=6379
SESSION_TIMEOUT=3600
COOKIE_SECURE=false
COOKIE_DOMAIN=localhost

# Internal Gateway Configuration
INTERNAL_GATEWAY_URL=http://internal-gateway:8081
JWT_VALIDATION_ENABLED=true
```

### Auth Gateway Implementation Checklist
- [ ] OAuth2 Authorization Code Flow implementation
- [ ] Session management with Redis
- [ ] CSRF protection with SameSite cookies
- [ ] Token refresh mechanism
- [ ] Security headers implementation
- [ ] CORS configuration
- [ ] Error handling with proper HTTP status codes
- [ ] Logout functionality with session cleanup
- [ ] Rate limiting for authentication endpoints
- [ ] Audit logging for security events
