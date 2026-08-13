# Marketplace Backend Architecture

## 1. Scope and Principles

This document defines the Phase 0 architecture for the marketplace platform foundation. The first milestone contains only API gateway, authentication, authorization, user management, RBAC, MySQL, and Docker infrastructure.

Product, shop, cart, order, payment, shipment, inventory, review, recommendation, notification, and search are intentionally outside this milestone.

The architecture favors independently deployable services, database ownership per service, explicit HTTP contracts, and a small operational footprint. It does not introduce service discovery, configuration server, messaging, Redis, Kubernetes, or distributed tracing infrastructure at this stage.

## 2. System Topology

```text
                         Client
                           |
                           v
                  +------------------+
                  |   API Gateway    |
                  | public entrypoint|
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
              v                         v
       +-------------+           +-------------+
       | Auth Service|           | User Service|
       |    :8081    |           |    :8082    |
       +------+------+           +------+------+ 
              |                         |
              v                         v
       +-------------+           +-------------+
       |  auth_db    |           |  user_db    |
       +-------------+           +-------------+
```

The API Gateway is the only public service in production. Auth Service and User Service are available only on the private Docker network. The gateway forwards requests using Docker service names, for example `http://auth-service:8081` and `http://user-service:8082`.

## 3. Service Boundaries

### API Gateway

The gateway is a Spring Cloud Gateway application and contains no domain business logic. Its responsibilities are:

- Route `/api/v1/auth/**` to Auth Service.
- Route `/api/v1/users/**` to User Service.
- Apply CORS policy at the edge.
- Create or propagate an `X-Correlation-Id` request identifier.
- Validate bearer access tokens before forwarding protected requests.
- Forward authenticated identity and authorities as trusted internal request headers.
- Convert gateway-level failures to the standard API error response.
- Leave a deliberate extension point for future rate limiting.

The gateway permits unauthenticated registration, login, refresh, and health endpoints. It rejects missing, malformed, expired, or invalid access tokens for protected routes before proxying. Services remain responsible for endpoint-level authorization.

### Auth Service

Auth Service is the sole owner of credentials and authentication sessions. Its responsibilities are registration, password hashing, login, token issuance, refresh-token rotation, logout, JWT generation, and JWT validation support.

It exposes these initial public APIs through the gateway:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `POST /api/v1/auth/logout`

Its database holds login identifiers, password hashes, refresh-token/session records, and minimal identity linkage required for authentication. It does not own profile data or RBAC data.

### User Service

User Service owns platform user records, profiles, account status, roles, permissions, and role assignment. It exposes these initial public APIs through the gateway:

- `GET /api/v1/users`
- `GET /api/v1/users/{id}`
- `GET /api/v1/users/me`
- `PATCH /api/v1/users/{id}`

The service prepares its API and persistence design for pagination, filtering, user status, role assignment, and permission lookup. It does not store passwords, issue tokens, or independently authenticate credentials.

## 4. Database Ownership and Data Model

Each service has its own MySQL database/schema and Flyway migration history. No service reads or writes another service's tables.

| Owner | Database | Initial owned data |
| --- | --- | --- |
| Auth Service | `auth_db` | authentication identity, password hash, refresh-token/session state |
| User Service | `user_db` | user profile, status, roles, permissions, user-role and role-permission relationships |

Both services use a stable user UUID as the cross-service identity reference. Registration is coordinated through an API contract, not a shared transaction or shared table. Because MySQL transactions cannot span service databases, registration must be designed as a compensating workflow: create authentication identity, create user record, and compensate or mark the identity inactive if the second operation fails. This is acceptable for the first milestone and keeps database ownership intact.

Flyway is the authoritative schema migration mechanism. Production configuration disables Hibernate schema generation (`ddl-auto=validate` or `none`); migrations are versioned per service under that service's `db/migration` directory.

## 5. Authentication and JWT Design

### Login and Refresh Flow

```text
Client -> Gateway -> Auth Service: credentials
Auth Service -> auth_db: verify login identifier and password hash
Auth Service -> Client: short-lived access token + long-lived refresh token

Client -> Gateway -> Auth Service: refresh token
Auth Service -> auth_db: verify active refresh-token record
Auth Service -> Client: new access token + rotated refresh token
```

Passwords use BCrypt with a configurable work factor. Plain-text passwords, access tokens, refresh tokens, JWT secrets, and authorization headers must never be logged.

Access tokens are short-lived JWTs, with configurable expiration. Refresh tokens are opaque, high-entropy values stored server-side only as hashes. A refresh operation rotates the refresh token: the submitted token is revoked and replaced atomically where possible. Logout revokes the associated refresh-token session. This permits logout and session invalidation without maintaining an access-token denylist.

JWTs use a symmetric signing key supplied only through `JWT_SECRET` for the initial milestone. A token contains only authorization-relevant, non-sensitive claims:

- `sub`: stable user UUID
- `iss`: configured issuer
- `aud`: configured marketplace API audience
- `iat`, `exp`, and `jti`
- `roles`: role names
- `authorities`: permission strings such as `user:read`

The gateway and User Service validate signature, issuer, audience, expiration, and required claims. The gateway validates requests at the edge; User Service validates again before trusting identity or authority headers. This defense-in-depth design prevents direct internal access or header spoofing from becoming authorization bypasses.

Symmetric signing is intentionally simple for three services. A future move to asymmetric signing and a published public key is appropriate when independently deployed services, third-party verifiers, or key rotation requirements justify its additional operational cost.

## 6. Authorization Flow

RBAC is stored and managed by User Service:

```text
User --< UserRole >-- Role --< RolePermission >-- Permission
```

Initial roles are `SUPER_ADMIN`, `ADMIN`, `SELLER`, and `CUSTOMER`. Permissions use the `resource:action` format, for example `user:read`, `user:update`, and `role:read`.

At login and refresh, Auth Service obtains the current role and permission snapshot from User Service through an internal API contract. It places this snapshot into the issued access token. The gateway converts validated JWT claims to trusted identity and authority context; User Service maps the context to Spring Security authorities. Endpoint and method access use permission checks such as `@PreAuthorize("hasAuthority('user:read')")`, not hard-coded role conditionals.

Authorization changes take effect for newly issued access tokens. The bounded delay is the access-token lifetime. This trade-off avoids a synchronous permission lookup for every request and keeps the gateway fast. For later high-risk administration operations, the design can add token-version checks or immediate session revocation without changing service database ownership.

## 7. Gateway Request Flow and Error Model

For each request, the gateway:

1. Accepts the request at a versioned `/api/v1/...` route.
2. Uses an incoming `X-Correlation-Id` or generates one, then forwards it to downstream services and logs it.
3. Applies CORS and route policy.
4. Validates bearer JWTs for protected routes.
5. Forwards the request to the target internal service using its Docker DNS name.
6. Maps gateway failures, including unknown routes and invalid tokens, to the common error contract.

Services use the same error envelope for validation, authentication, authorization, domain, and unexpected errors:

```json
{
  "timestamp": "2026-08-13T15:00:00Z",
  "status": 400,
  "code": "INVALID_REQUEST",
  "message": "Invalid request",
  "path": "/api/v1/users"
}
```

Responses never expose stack traces, internal URLs, SQL details, passwords, tokens, or secrets. Correlation IDs are used in logs to diagnose errors without disclosing internals to clients.

## 8. Inter-Service Communication

The initial communication style is synchronous HTTP over the private Docker network. It is sufficient for the limited scope and easier to reason about than introducing a broker.

Internal endpoints must be explicitly separated from public API routes where needed, for example an Auth Service request to retrieve a user's authorization snapshot from User Service. These calls use Docker service names, timeouts, and service authentication appropriate to the implementation phase. They do not use `localhost` or direct database access.

At this phase, service locations are configured with environment variables. Static Docker Compose service DNS avoids premature introduction of service discovery or a configuration server.

## 9. Docker Architecture and Configuration

Docker Compose runs four containers on a private bridge network:

- `api-gateway`
- `auth-service`
- `user-service`
- `mysql`

MySQL exposes two isolated databases, `auth_db` and `user_db`, with separate application credentials in the preferred setup. The database port need not be published to the host for normal application operation. Only the gateway publishes a host port; internal service ports remain private.

Container health checks gate dependent service startup where Compose support permits. Application health endpoints support operational readiness checks. Each service receives configuration via environment variables, including `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, token lifetimes, and internal service URLs. `.env.example` documents required variables; `.env` is excluded from version control.

## 10. Proposed Project Structure

```text
microservice_project/
├── ARCHITECTURE.md
├── pom.xml
├── docker-compose.yml
├── .env.example
├── api-gateway/
│   ├── pom.xml
│   └── src/
├── auth-service/
│   ├── pom.xml
│   └── src/
├── user-service/
│   ├── pom.xml
│   └── src/
└── docs/
    ├── architecture.md
    ├── authentication.md
    ├── authorization.md
    ├── database.md
    ├── docker.md
    ├── api.md
    └── adr/
```

Each Spring Boot service follows controller, DTO, service, repository, entity, security, configuration, exception, and migration package boundaries. Controllers accept and return DTOs, use Jakarta Validation, and delegate to services. Services use constructor injection. Repositories remain persistence-focused, and JPA entities are never returned directly from APIs.

## 11. Security Model

- TLS terminates at the production ingress or gateway deployment boundary; local Compose may use HTTP only for development.
- The gateway has the public surface; internal services are network-private.
- Authentication is centralized in Auth Service.
- Authorization data is centralized in User Service and enforced by Spring Security permissions.
- Password and refresh-token storage uses one-way hashes.
- Secrets are environment-provided and excluded from source control and logs.
- Input validation, consistent error handling, least-privilege database users, and correlation IDs are baseline controls.
- The gateway and resource service both validate access tokens before privileged actions.

## 12. Important Trade-Offs and Decisions

| Decision | Benefit | Cost / Mitigation |
| --- | --- | --- |
| Separate service databases | Clear ownership and independent evolution | Registration is eventually consistent; use compensation rather than shared transactions |
| Gateway JWT validation plus service revalidation | Fast rejection and defense in depth | Duplicate validation logic; keep token-validation rules in a shared, versioned library only if duplication becomes material |
| Permission snapshot in JWT | No permission lookup on every request | Role changes wait until access token expiry; keep access tokens short-lived |
| Opaque hashed refresh tokens | Supports logout, rotation, and revocation | Requires database access during refresh; refresh requests are low volume |
| Symmetric JWT key initially | Simple configuration and deployment | All validators need the secret; migrate to asymmetric keys when independent key distribution is needed |
| HTTP over Docker network | Minimal infrastructure and clear contracts | No asynchronous resilience for cross-service registration; add events only when a concrete workflow needs them |
| One MySQL container with separate databases initially | Low operational overhead and no local MySQL requirement | It is not independent database availability; split into separate instances when scaling or isolation demands it |

## 13. Phase Boundaries

This document is the complete Phase 0 deliverable. It makes no application, Maven, Docker Compose, migration, or configuration implementation changes. Those artifacts begin only after explicit approval to start Phase 1.
