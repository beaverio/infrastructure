# 🏗️ BEAVER MICROSERVICES ARCHITECTURE RECAP

## 🎯 What We're Building: Multi-Tenant SaaS with Secure Workspace Isolation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           🌐 CLIENT (React/SPA)                              │
│                        ❌ NO TOKENS IN BROWSER                              │
│                        ✅ HTTP-ONLY COOKIES ONLY                           │
└─────────────────┬───────────────────────────────────────────────────────────┘
                  │
                  │ 🍪 HTTP-Only Cookies
                  │ (app_session)
                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        🛡️ BFF (Backend For Frontend)                        │
│                                                                             │
│  • Session Management (Redis)                                              │
│  • Token Exchange with Keycloak                                            │
│  • API Proxy → Gateway                                                     │
│  • Gets User Context from Identity Service                                 │
└─────────────────┬───────────────────────────────────────────────────────────┘
                  │
                  │ 🎟️ Workspace-Scoped JWT
                  │ Bearer: eyJ... (userId, workspaceId, roles)
                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      🚪 GATEWAY (Spring Cloud Gateway)                      │
│                                                                             │
│  • JWT Validation Only                                                     │
│  • Rate Limiting                                                           │
│  • Service Routing                                                         │
│  • NO Header Enrichment (Services Read JWT Directly)                      │
└─────────────────┬───────────────────────────────────────────────────────────┘
                  │
                  │ 🎟️ Forward JWT As-Is
                  │ Bearer: eyJ... (unchanged)
                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          🎯 MICROSERVICES                                   │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Identity   │  │Transactions │  │ Reporting   │  │   Billing   │        │
│  │  Service    │  │   Service   │  │   Service   │  │   Service   │        │
│  │             │  │             │  │             │  │             │        │
│  │ 👤 Users    │  │ 💰 Txns     │  │ 📊 Charts   │  │ 💳 Invoices │        │
│  │ 🏢 Workspaces│  │ 🏦 Accounts │  │ 📈 Analytics│  │ 💵 Payments │        │
│  │ 👥 Members  │  │ 🔄 Transfers│  │ 📋 Reports  │  │ 📄 Receipts │        │
│  │             │  │             │  │             │  │             │        │
│  │ 🔓 Read JWT │  │ 🔓 Read JWT │  │ 🔓 Read JWT │  │ 🔓 Read JWT │        │
│  │ Claims      │  │ Claims      │  │ Claims      │  │ Claims      │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────┬───────────────────────────────────────────────────────────┘
                  │
                  │ 🔒 RLS Enforced SQL
                  │ WHERE workspace_id = current_setting('app.workspace_id')
                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       🗄️ POSTGRES (Row-Level Security)                      │
│                                                                             │
│  • Workspace Isolation at DB Level                                         │
│  • RLS Policies: current_setting('app.workspace_id')                       │
│  • Services Set Session Var from JWT Claims                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Authentication Flow Breakdown

### 1️⃣ **LOGIN PHASE**
```
Client → BFF → Keycloak → BFF → Redis
  ❌        🎟️         🔑      🍪
```
- Client hits `/app` with no session
- BFF redirects to Keycloak OIDC
- User authenticates, BFF gets tokens
- Session stored in Redis, cookie set

### 2️⃣ **USER CONTEXT RETRIEVAL**
```
BFF → Gateway → Identity → PostgreSQL → Identity → Gateway → BFF
  🎟️      🚪        👤           🗄️          📋       🚪      📝
```
- BFF calls `/identity/users/me` with Keycloak access token
- Identity service queries: `SELECT user, lastWorkspaceId, roles WHERE id = jwt.sub`
- Returns user profile + lastWorkspaceId + roles to BFF

### 3️⃣ **TOKEN EXCHANGE**
```
BFF → Keycloak → BFF
  🎫      🏭      🎟️
```
- BFF exchanges generic token for workspace-scoped JWT
- Custom claims: `{userId, workspaceId, roles}`
- JWT contains: `sub`, `workspaceId`, `roles`, `aud`
- Short-lived (5 minutes) for security

### 4️⃣ **API CALLS**
```
Client → BFF → Gateway → Services → Database
  🍪      🎟️      🎟️       🔓       💾
```
- Client sends cookie-only requests
- BFF proxies with workspace-scoped JWT
- Gateway validates JWT and forwards as-is
- Services extract claims directly from JWT
- Services set `app.workspace_id` session var for RLS

## 🏗️ **COMPONENTS TO BUILD**

### 🛡️ **BFF Service** (New)
- **Tech**: Spring Boot + WebFlux
- **Responsibilities**:
  - Session management (Redis)
  - OIDC integration (Keycloak)
  - Get user context from identity service
  - Token exchange with custom claims
  - API proxying

### 🚪 **Gateway Service** (New)
- **Tech**: Spring Cloud Gateway
- **Responsibilities**:
  - JWT validation only
  - Service routing
  - Rate limiting
  - NO header enrichment (services read JWT)

### 🎯 **Microservices** (New)
- **Identity Service**: Users, workspaces, memberships, `/me` endpoint
- **Transaction Service**: Financial transactions
- **Reporting Service**: Analytics and reports
- **Billing Service**: Invoices and payments
- **Auth Pattern**: All services read JWT claims directly

### 🗄️ **Database Layer**
- **Tech**: PostgreSQL with RLS
- **Isolation**: Services set `app.workspace_id` from JWT claims
- **Policies**: `WHERE workspace_id = current_setting('app.workspace_id')`

### 🔐 **Auth Infrastructure**
- **Keycloak**: OIDC provider + token exchange + custom claims
- **Redis**: Session storage + optional roles caching

## 🎯 **KEY FLOW DIFFERENCES**

✅ **BFF Gets User Context First**: Calls identity service to get lastWorkspaceId + roles  
✅ **Token Exchange with Custom Claims**: BFF mints workspace-scoped JWT via Keycloak  
✅ **No Header Enrichment**: Gateway just validates and forwards JWT  
✅ **Services Read JWT**: Each service extracts claims directly from Bearer token  
✅ **RLS Session Variables**: Services set `app.workspace_id` from JWT for database isolation  

## 🚀 **DEPLOYMENT STRATEGY**

```
┌─────────────────────────────────────────────────────────────┐
│                    🌥️ AWS CLOUD                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │     EKS     │  │    RDS      │  │ ElastiCache │        │
│  │             │  │             │  │             │        │
│  │ Kubernetes  │  │ PostgreSQL  │  │    Redis    │        │
│  │ Services    │  │             │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐                         │
│  │     ALB     │  │  Route 53   │                         │
│  │             │  │             │                         │
│  │Load Balancer│  │     DNS     │                         │
│  └─────────────┘  └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

**This is a clean, JWT-native multi-tenant architecture!** 🚀

