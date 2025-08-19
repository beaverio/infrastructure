# Authentication Flow Architecture

## Overview
This document describes the OAuth2/OpenID Connect authentication flow using Keycloak as the identity provider in a microservices architecture with Auth Gateway pattern for session-to-JWT orchestration.

## Authentication Flow Diagram

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant AuthGW as 🔐 Auth Gateway
    participant Keycloak as 🔐 Keycloak
    participant Provider as 🌐 OAuth Provider (Google/Apple)
    participant Redis as ⚡ Session Store
    participant InternalGW as 🚪 Internal Gateway
    participant Identity as 👤 Identity Service
    participant Transactions as 💰 Transactions Service
    participant IdentityDB as 🗄️ Identity DB
    participant TransactionsDB as 🗄️ Transactions DB

    Note over User,TransactionsDB: 1. OAuth2 Sign In Flow with Provider Selection
    User->>AuthGW: Access protected resource OR click login
    AuthGW->>Redis: Check for existing session
    Redis-->>AuthGW: No session found
    AuthGW-->>User: Redirect to Keycloak realm
    
    User->>Keycloak: Access Keycloak login page
    Keycloak-->>User: Show provider selection (Google, Apple, etc.)
    User->>Keycloak: Select Google OAuth
    Keycloak-->>User: Redirect to Google OAuth
    
    User->>Provider: Login with Google credentials
    Provider-->>User: Redirect to Keycloak callback with auth code
    User->>Keycloak: Google auth code + user profile
    Keycloak->>Keycloak: Create/update user from Google profile
    Keycloak-->>User: Redirect to Auth Gateway with authorization code
    
    Note over AuthGW,TransactionsDB: 2. Authorization Code Exchange & Context Building
    User->>AuthGW: Callback with Keycloak authorization code
    AuthGW->>Keycloak: Exchange auth code for tokens
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
    AuthGW-->>User: Set HTTP-only session cookie

    Note over User,TransactionsDB: 3. API Call with Workspace Context
    User->>AuthGW: GET /api/transactions/recent (session cookie)
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
    Transactions->>TransactionsDB: SELECT * FROM transactions ORDER BY date DESC LIMIT 10
    Note right of TransactionsDB: RLS: WHERE workspace_id = current_setting('app.workspace_id')
    TransactionsDB-->>Transactions: Workspace-filtered results
    Transactions-->>InternalGW: Transaction data
    InternalGW-->>AuthGW: Pass through response
    AuthGW-->>User: JSON response

    Note over User,TransactionsDB: 4. Session Expiry Handling
    User->>AuthGW: API call after session expires
    AuthGW->>Redis: Check session
    Redis-->>AuthGW: Session expired or not found
    AuthGW-->>User: 401 Unauthorized + redirect to login
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

## Session Management

### Auth Gateway Session Strategy
- **Server-side Sessions**: Stored in Redis for scalability
- **HTTP-only Cookies**: Prevent XSS attacks
- **Secure Flag**: HTTPS-only transmission
- **SameSite**: CSRF protection
- **Session Timeout**: Configurable expiration

### Redis Session Store
- **Session Data**: User ID, current workspace, refresh token, roles cache
- **TTL**: Idle and absolute expiration times
- **Connection**: Auth Gateway connects to Redis for session operations

### Session Flow
1. User accesses a protected resource.
2. Auth Gateway checks for a valid session in Redis.
3. If no valid session, respond with 401 Unauthorized and redirect to login.
4. After successful login, store session data in Redis with access and refresh tokens.
5. Set HTTP-only, Secure, SameSite cookie in the user's browser.
6. For subsequent requests, Auth Gateway validates the session using the cookie.
7. If the access token is expired, use the refresh token to obtain a new access token.
8. Update the session in Redis with the new access token.

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

## Authorization Matrix
| Role | Identity Service | Transactions Service | Internal Gateway Access |
|------|-----------------|---------------------|----------------|
| **user** | Read own profile | Read own transactions | Basic routes |
| **manager** | Read team profiles | Read team transactions | Manager routes |
| **admin** | Full access | Full access | All routes |

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
```
