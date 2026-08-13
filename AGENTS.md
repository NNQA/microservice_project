# Marketplace Backend — Agent Instructions

## 1. Project Overview

This repository is a Java Spring Boot microservices backend for an online marketplace.

The system is designed to evolve into a marketplace similar to Shopee.

The architecture must support independent services, independent deployment, service-owned databases, and future horizontal scaling.

---

# 2. Technology Stack

Use:

* Java 21+
* Spring Boot 3.x
* Spring Cloud
* Spring Cloud Gateway
* Spring Security
* Spring Data JPA
* MySQL 8+
* Maven
* Docker
* Docker Compose
* Flyway
* JUnit
* Testcontainers when appropriate

Do not require local MySQL.

Infrastructure must run through Docker.

---

# 3. Current Scope

The current implementation scope is intentionally small.

Focus only on:

* API Gateway
* Authentication
* Authorization
* User management
* RBAC
* MySQL infrastructure
* Docker infrastructure

Do NOT implement these domains yet:

* Product
* Shop
* Cart
* Order
* Payment
* Shipment
* Inventory
* Review
* Recommendation
* Notification
* Search

These domains may be added later.

---

# 4. Microservice Architecture

The initial services are:

```text
api-gateway
auth-service
user-service
```

Architecture:

```text
Client
   |
   v
API Gateway
   |
   +-------------------+
   |                   |
   v                   v
Auth Service       User Service
   |                   |
   v                   v
Auth Database       User Database
```

The API Gateway is the public entry point.

Clients should not directly communicate with internal services in the production architecture.

---

# 5. Service Boundaries

## API Gateway

Responsibilities:

* Routing
* Authentication filter
* CORS
* Correlation ID
* Global gateway error handling
* Future rate limiting
* Request forwarding

The Gateway must NOT contain business logic.

---

## Auth Service

Responsibilities:

* Registration
* Login
* Logout
* Access token
* Refresh token
* Password hashing
* JWT generation
* JWT validation
* Authentication logic

Authentication must be centralized here.

---

## User Service

Responsibilities:

* User profile
* User information
* User status
* User management
* Role assignment
* Permission information

Do not duplicate authentication logic inside User Service.

---

# 6. Database Ownership

Each service owns its database.

A service MUST NOT directly access another service's database.

Preferred architecture:

```text
Auth Service
    |
    v
Auth DB

User Service
    |
    v
User DB
```

Services communicate through APIs or events.

Never solve service communication by sharing database tables.

---

# 7. Authentication

Use:

```text
JWT Access Token
+
Refresh Token
```

Access tokens should be short-lived.

Refresh tokens should be longer-lived.

Passwords must never be stored in plain text.

Use a secure password hashing algorithm such as:

* BCrypt
* Argon2

Do not log:

* Passwords
* Access tokens
* Refresh tokens
* Secrets

---

# 8. Authorization

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

Permissions use the format:

```text
resource:action
```

Examples:

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

Do not hard-code authorization logic like:

```java
if (role.equals("ADMIN")) {
}
```

Prefer Spring Security authorization mechanisms such as:

```java
@PreAuthorize("hasAuthority('user:read')")
```

---

# 9. API Convention

Use versioned REST APIs:

```text
/api/v1
```

Examples:

```text
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout

GET    /api/v1/users
GET    /api/v1/users/{id}
GET    /api/v1/users/me
PATCH  /api/v1/users/{id}
```

Use consistent HTTP status codes.

---

# 10. Configuration

Use environment variables for:

* Database credentials
* Database host
* Database port
* JWT secret
* Token configuration
* Service URLs

Never hard-code secrets.

Provide:

```text
.env.example
```

Never commit `.env`.

---

# 11. Docker

Infrastructure must work through Docker Compose.

MySQL must run inside Docker.

Services must communicate through Docker networking.

Inside Docker, use service names instead of localhost.

Example:

```text
http://auth-service:8081
http://user-service:8082
```

Do not use:

```text
http://localhost:8081
```

for internal service-to-service communication inside Docker.

---

# 12. Database Migration

Use Flyway.

Do not rely on:

```text
spring.jpa.hibernate.ddl-auto=create
```

for production database schema management.

Database changes must be version-controlled.

Example:

```text
db/migration/
├── V1__create_users.sql
├── V2__create_roles.sql
└── V3__create_permissions.sql
```

---

# 13. Code Standards

Use:

* Constructor injection
* DTOs
* Jakarta Validation
* Global exception handling
* Service layer
* Repository layer
* Clean separation of responsibilities

Avoid:

* Field injection
* Business logic in controllers
* Business logic in repositories
* Exposing JPA entities directly
* Static global state
* Hard-coded secrets

Use Lombok only when it clearly improves readability.

---

# 14. Testing

Each service should have appropriate:

### Unit tests

For:

* Services
* Authentication
* JWT
* Authorization

### Integration tests

For:

* REST APIs
* Database interaction
* Gateway routing

Use Testcontainers when integration tests require real infrastructure.

---

# 15. Documentation

Architecture decisions should be documented.

Expected documentation:

```text
docs/
├── architecture.md
├── authentication.md
├── authorization.md
├── database.md
├── docker.md
└── api.md
```

Important architectural decisions may use:

```text
docs/adr/
```

---

# 16. Development Workflow

Do NOT implement the entire project at once.

Work phase by phase.

Before implementation:

1. Analyze the requirement.
2. Analyze architecture.
3. Identify trade-offs.
4. Explain important decisions.
5. Wait for confirmation when requested.

When implementing a phase:

1. Implement only that phase.
2. Do not silently expand scope.
3. Run tests.
4. Run build.
5. Check relevant Docker configuration.
6. Report changed files.
7. Suggest a Git commit message.

---

# 17. Scope Discipline

If a request would introduce a new domain, do not automatically implement it.

For example, if currently working on Authentication, do not start implementing:

* Product
* Shop
* Order
* Payment

unless explicitly requested.

---

# 18. Architecture Changes

If the existing architecture has a problem:

1. Explain the problem.
2. Explain the impact.
3. Propose alternatives.
4. Recommend one.
5. Wait for confirmation before making a significant architectural change.

Do not silently rewrite established architecture.

---

# 19. Important Principle

Prefer simple, understandable architecture over premature complexity.

This is a learning and production-oriented project.

Do not introduce:

* Kafka
* Redis
* Kubernetes
* Config Server
* Service Discovery
* Distributed tracing infrastructure
* Event sourcing

unless there is a clear requirement.

The architecture should remain extensible without prematurely implementing every infrastructure component.

---

# 20. Agent Behavior

Always inspect the existing repository before creating files.

Never assume a file does not exist.

Never overwrite existing work without checking it first.

Before making significant changes, explain what will change when the user has requested planning first.

Keep changes focused and reviewable.

Do not move to the next development phase unless explicitly instructed.
