# Docker Container Architecture

This document outlines the local container setup for the Beaver microservices architecture.

## Container Network Topology

```mermaid
graph LR
    subgraph "Internet / Host Machine"
        Browsers[🌐 Browsers]
        DevTools[🛠️ Postman, etc.]
    end
    
    subgraph "Private Container Network"
        AuthGateway["🔐 auth-gateway<br/>8080:8080<br/>EXPOSED FOR DEVELOPMENT"]
        
        Keycloak[🔐 keycloak<br/>8090:8080]
        KeycloakDB[(🗄️ keycloak-db<br/>5433:5432<br/>PostgreSQL)]
        AuthRedis[(📦 auth-redis<br/>6379:6379)]

        InternalGateway[🚪 internal-gateway<br/>:8081<br/>INTERNAL ONLY]

        Identity[👤 identity-service<br/>:8082<br/>INTERNAL ONLY]
        Transactions[💰 transactions-service<br/>:8083<br/>INTERNAL ONLY]

        IdentityDB[(🗄️ identity-db<br/>5434:5432<br/>PostgreSQL)]
        TransactionsDB[(🗄️ transactions-db<br/>5435:5432<br/>PostgreSQL)]
    end
    
    Browsers --> AuthGateway
    DevTools --> AuthGateway
    
    AuthGateway --> Keycloak
    AuthGateway --> InternalGateway
    AuthGateway --> AuthRedis
    
    InternalGateway --> Identity
    InternalGateway --> Transactions
    
    Identity --> IdentityDB
    Transactions --> TransactionsDB
    
    Keycloak --> KeycloakDB
    
    classDef internet fill:#ffebee,color:#c62828
    classDef auth fill:#e8f5e8,color:#2e7d32
    classDef internal fill:#fff3e0,color:#e65100
    classDef database fill:#bbdefb,color:#0d47a1
    classDef cache fill:#ffcdd2,color:#b71c1c
    
    class Browsers,DevTools internet
    class AuthGateway auth
    class Keycloak,InternalGateway,Identity,Transactions internal
    class IdentityDB,TransactionsDB,KeycloakDB database
    class AuthRedis cache
```

## Network Security (Authentication Gateway Architecture)

### Single Private Network
- **Docker**: All containers run in one custom Docker bridge network with internal DNS resolution
- **Container isolation**: Only containers in the same network can communicate with each other
- **No external access**: Internal containers have no published ports to host machine
- **DNS resolution**: Containers communicate using container names (e.g., `http://keycloak:8090`)
(AWS equivalent: VPC with private subnets)

### Auth Gateway (Authentication Orchestration Layer)
- **Docker**: Single container with published port mapping `8080:8080` to host machine
- **External access**: Only this container is reachable from outside Docker network
- **Authentication role**: Converts session cookies to JWT tokens and manages auth state
- **Pass-through pattern**: Forwards authenticated requests to Internal Gateway
- **Session management**: Uses Redis container for distributed session storage
- **Auth boundary**: Separates public authentication (cookies) from private authentication (JWT)
(AWS equivalent: Application Load Balancer + ECS Task in public subnet)

### Internal Gateway (Service Routing Hub)
- **Docker**: Container with no published ports - internal access only via `internal-gateway:8081`
- **Routing responsibility**: Receives JWT-authenticated requests from Auth Gateway and routes to services
- **Service discovery**: Uses Docker DNS to forward requests to `identity:8082`, `transactions:8083`, etc.
- **Token validation**: Validates JWT tokens before forwarding to downstream services
- **Path-based routing**: Routes based on URL patterns (e.g., `/identity/*` → Identity Service)

### Business Services (Private Microservices)
- **Docker**: Containers with no published ports - only accessible within Docker network
- **Service discovery**: Containers find each other using Docker's built-in DNS (container names)
- **Database isolation**: Each service has dedicated PostgreSQL container with unique internal port
- **JWT consumption**: Services receive validated JWT tokens with user/workspace context
- **Network segmentation**: Cannot be accessed from host machine or external networks
(AWS equivalent: ECS Tasks in private subnets + RDS instances + ElastiCache)

## Request Flow Architecture

### Client Request Journey
```
Client Request (🍪 session cookie)
    ↓
┌─────────────────────────────────────────┐
│           Auth Gateway :8080          │
│    Authentication Orchestrator         │
│                                         │
│  1. Extract session from Redis          │
│  2. Get JWT from session                │
│  3. Add Authorization header            │
│  4. Forward request to Internal Gateway  │
└─────────────────────────────────────────┘
    ↓ (🎟️ JWT Bearer token)
┌─────────────────────────────────────────┐
│         Internal Gateway :8081         │
│       Service Router                   │
│                                         │
│  1. Validate JWT token                  │
│  2. Route by path pattern               │
│  3. Forward to appropriate service      │
└─────────────────────────────────────────┘
    ↓ (🎟️ Forward JWT unchanged)
┌─────────────────────────────────────────┐
│    Microservices :8082, :8083, etc.    │
│       Business Logic                    │
│                                         │
│  1. Extract user context from JWT       │
│  2. Set database session variables      │
│  3. Execute business logic              │
│  4. Return response                     │
└─────────────────────────────────────────┘
```

### Response Journey
```
Service Response (JSON)
    ↓
Internal Gateway (passes through unchanged)
    ↓  
Auth Gateway (passes through unchanged)
    ↓
Client receives identical response
```

## Development vs Production

### Development (Local Docker)
```
Host Machine → Auth Gateway :8080 → Internal Gateway :8081 → Services :808X
    🍪              🎟️           🚪            🔓
 (session)        (JWT)      (route)     (business logic)
```

**No Load Balancer Needed Locally:**
- Auth Gateway port 8080 exposed to localhost for testing
- Direct access: `curl http://localhost:8080/api/identity/users/me`
- Auth Gateway orchestrates: cookie → JWT → forward to internal gateway
- Simplified for development efficiency

### Production (Cloud)
```
Internet → Load Balancer → Auth Gateway → Internal Gateway → Services
    🌐           🔀           🎟️      🚪        🔓
               (SSL)      (orchestrate) (route) (execute)
```


## Container Configuration

### Individual Service Dockerfiles
Each service will have its own Dockerfile and be built/managed individually through IntelliJ Services:

**Service Structure:**
```
services/
├── bff-web/
│   ├── Dockerfile
│   ├── src/
│   └── application-local.yml
├── gateway/
│   ├── Dockerfile
│   ├── src/
│   └── application-local.yml
├── identity-service/
│   ├── Dockerfile
│   ├── src/
│   └── application-local.yml
└── transactions-service/
    ├── Dockerfile
    ├── src/
    └── application-local.yml
```

### Environment Variables Strategy
Each service uses Spring Boot profiles and environment-specific configuration:
- `application-local.yml` - Local development configuration
- `application-test.yml` - Testing environment
- `application-prod.yml` - Production (not in repo)

**Example service configuration patterns:**
```yaml
# Database connection (each service has dedicated DB)
spring:
  datasource:
    url: jdbc:postgresql://identity-db:5434/identity_db
    username: identity_user
    password: identity_password

# Redis connection (Auth Gateway only)
  session:
    store-type: redis
  data:
    redis:
      host: auth-redis
      port: 6379

# Keycloak configuration
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8090/realms/dev

# Internal service discovery
gateway:
  url: http://internal-gateway:8081
```

## IntelliJ Services Management

### Infrastructure Containers (Docker)
These will be standalone Docker containers you start manually:
- **keycloak** - Authentication server (port 8090:8080)
- **keycloak-db** - PostgreSQL for Keycloak (port 5433:5432)
- **auth-redis** - Session storage for Auth Gateway (port 6379:6379)
- **identity-db** - PostgreSQL for Identity service (port 5434:5432)
- **transactions-db** - PostgreSQL for Transactions service (port 5435:5432)

### Application Services (IntelliJ)
These will be Spring Boot applications managed through IntelliJ Services:
- **bff-web** - Built from Dockerfile, exposed port 8080:8080
- **gateway** - Built from Dockerfile, internal port 8081
- **identity-service** - Built from Dockerfile, internal port 8082
- **transactions-service** - Built from Dockerfile, internal port 8083

## Development Workflow

### 1. Start Infrastructure Containers (Manual Docker Commands)
Start the required infrastructure containers first:
```bash
# Create custom Docker network
docker network create beaver-network

# Start databases and cache
docker run -d --name keycloak-db --network beaver-network \
  -e POSTGRES_DB=keycloak_dev -e POSTGRES_USER=keycloak_user -e POSTGRES_PASSWORD=keycloak_password \
  -p 5433:5432 postgres:16

docker run -d --name identity-db --network beaver-network \
  -e POSTGRES_DB=identity_db -e POSTGRES_USER=identity_user -e POSTGRES_PASSWORD=identity_password \
  -p 5434:5432 postgres:16

docker run -d --name transactions-db --network beaver-network \
  -e POSTGRES_DB=transactions_db -e POSTGRES_USER=transactions_user -e POSTGRES_PASSWORD=transactions_password \
  -p 5435:5432 postgres:16

docker run -d --name auth-redis --network beaver-network \
  -p 6379:6379 redis:7-alpine

docker run -d --name keycloak --network beaver-network \
  -e KC_DB=postgres -e KC_DB_URL=jdbc:postgresql://keycloak-db:5432/keycloak_dev \
  -e KC_DB_USERNAME=keycloak_user -e KC_DB_PASSWORD=keycloak_password \
  -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin_password \
  -p 8090:8080 quay.io/keycloak/keycloak:25.0 start-dev
```

### 2. Build and Run Services via IntelliJ Services
1. **Configure IntelliJ Services**: Add each service to IntelliJ's Services panel
2. **Build Dockerfiles**: Each service builds from its individual Dockerfile
3. **Network Configuration**: Ensure all services join the `beaver-network`
4. **Start Order**: Start services in dependency order:
   - Infrastructure containers (already running)
   - **gateway** (internal routing)
   - **identity-service** (user context)
   - **transactions-service** (business logic)
   - **bff-web** (public endpoint)

### 3. Access and Testing
- **Application Access**: http://localhost:8080 (Auth Gateway only)
- **API Testing**: Use Postman/curl against Auth Gateway endpoints
- **Database Access**: Connect via exposed ports for development (5433, 5434, 5435)
- **Service Logs**: Monitor via IntelliJ Services panel

### 4. Development Cycle
1. **Code Changes**: Edit source code in IntelliJ
2. **Hot Reload**: Services rebuild automatically on file changes
3. **Service Restart**: Use IntelliJ Services to restart individual containers
4. **Network Communication**: Services discover each other via Docker DNS
