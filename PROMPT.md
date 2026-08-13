# Marketplace Backend — Project Specification

## Objective

Build a production-oriented Java Spring Boot microservices backend for an online marketplace.

The first milestone focuses only on the platform foundation:

- API Gateway
- Authentication
- Authorization
- RBAC
- User Service
- MySQL
- Docker

Do not implement business domains such as Product, Shop, Order, Payment, Cart, Shipment, Inventory, Review, Recommendation, Notification, or Search yet.

---

# Technology

Use:

- Java 21+
- Spring Boot 3.x
- Spring Cloud
- Spring Cloud Gateway
- Spring Security
- Spring Data JPA
- MySQL 8+
- Maven
- Docker
- Docker Compose
- Flyway
- JUnit
- Testcontainers where appropriate

---

# Services

Create these services:

```text
api-gateway
auth-service
user-service
```

Expected architecture:

```text
                    Client
                       |
                       v
              +----------------+
              |  API Gateway   |
              +-------+--------+
                      |
             +--------+--------+
             |                 |
             v                 v
       Auth Service       User Service
             |                 |
             v                 v
          Auth DB            User DB
```

---

# API Gateway

Use Spring Cloud Gateway.

Responsibilities:

- Route requests
- Validate JWT
- CORS
- Correlation ID
- Global gateway error handling
- Prepare architecture for future rate limiting

Routes:

```text
/api/v1/auth/**  -> auth-service
/api/v1/users/** -> user-service
```

Gateway must not contain business logic.

---

# Authentication

Use:

```text
JWT Access Token
+
Refresh Token
```

Implement:

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

Passwords must be securely hashed.

Do not store plain-text passwords.

---

# Authorization

Use RBAC.

Relationship:

```text
User
 |
 +-- UserRole
       |
       v
      Role
       |
       +-- RolePermission
                |
                v
            Permission
```

Initial roles:

```text
SUPER_ADMIN
ADMIN
SELLER
CUSTOMER
```

Example permissions:

```text
user:read
user:create
user:update
user:delete
user:ban

role:read
role:create
role:update
role:delete

permission:read
```

Use Spring Security method-level authorization where appropriate.

Example:

```java
@PreAuthorize("hasAuthority('user:read')")
```

---

# User Service

Implement basic user management.

Expected endpoints:

```text
GET   /api/v1/users
GET   /api/v1/users/{id}
GET   /api/v1/users/me
PATCH /api/v1/users/{id}
```

Prepare the architecture for:

- User status
- Role assignment
- Permission lookup
- Pagination
- Filtering

---

# Database

Use MySQL.

MySQL must run inside Docker.

Do not require MySQL installed on the host machine.

Each service owns its database.

Use Flyway for schema migration.

Do not use Hibernate schema generation as the production migration strategy.

---

# Docker

Create:

```text
docker-compose.yml
.env.example
```

Docker infrastructure should provide:

```text
API Gateway
Auth Service
User Service
MySQL
```

Use a private Docker network.

Services communicate using Docker service names.

Example:

```text
auth-service:8081
user-service:8082
```

Do not use localhost for internal Docker communication.

---

# Configuration

Use environment variables for:

```text
DB_HOST
DB_PORT
DB_NAME
DB_USERNAME
DB_PASSWORD
JWT_SECRET
```

Never commit secrets.

---

# Error Handling

Use a consistent API error response:

```json
{
  "timestamp": "...",
  "status": 400,
  "code": "INVALID_REQUEST",
  "message": "Invalid request",
  "path": "/api/v1/users"
}
```

Do not expose stack traces to clients.

---

# Testing

Implement:

- Authentication unit tests
- JWT tests
- Authorization tests
- User service tests
- API integration tests
- Gateway routing tests

Use Testcontainers where useful.

---

# Documentation

Create:

```text
docs/
├── architecture.md
├── authentication.md
├── authorization.md
├── database.md
├── docker.md
└── api.md
```

---

# Development Phases

## Phase 0 — Architecture

Analyze and document:

- Service boundaries
- Database ownership
- Authentication flow
- Authorization flow
- JWT design
- Gateway flow
- Docker architecture
- Project structure
- Inter-service communication
- Security model
- Important trade-offs

Create:

```text
ARCHITECTURE.md
```

Do NOT implement application code.

Stop after Phase 0.

---

## Phase 1 — Scaffold

Create:

- Maven parent configuration
- Spring Boot services
- API Gateway
- Auth Service
- User Service
- Basic project structure
- Docker configuration
- Environment configuration

All applications must compile.

---

## Phase 2 — Infrastructure

Implement:

- Docker Compose
- MySQL
- Docker network
- Health checks
- Environment configuration
- Flyway

Verify:

```bash
docker compose up -d
```

---

## Phase 3 — Authentication

Implement:

- Registration
- Login
- Logout
- JWT
- Refresh token
- Password hashing
- Authentication tests

---

## Phase 4 — Authorization

Implement:

- Roles
- Permissions
- UserRole
- RolePermission
- Spring Security authorization
- Method-level authorization
- Authorization tests

---

## Phase 5 — User Service

Implement:

- User CRUD
- Current user
- User status
- Role management
- Permission lookup
- Pagination/filter foundation

---

## Phase 6 — Gateway Security

Implement:

- JWT validation
- Routing
- CORS
- Correlation ID
- Error handling

Test complete flow:

```text
Client
  |
  v
Gateway
  |
  v
Auth/User Service
  |
  v
Database
```

---

# Important Rule

Do NOT implement all phases at once.

Start with Phase 0 only.

After completing Phase 0:

1. Create `ARCHITECTURE.md`.
2. Summarize the architecture.
3. List important trade-offs.
4. List files created.
5. STOP.

Wait for explicit instruction before continuing to Phase 1.
