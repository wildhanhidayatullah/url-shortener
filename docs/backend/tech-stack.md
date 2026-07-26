# Technology Stack

## A. Overview

This document records the technology decisions for the project.

The primary goals are:

- Keep the dependency graph small.
- Prefer Go standard library whenever practical.
- Choose mature and production-proven technologies.
- Favor explicit behavior over framework magic.
- Keep the architecture easy to understand and maintain.

## B. Technologies

### B.1. Runtime & Language

#### Go (Minimum Supported Version: **1.26**)

#### Rationale

- Excellent concurrency model.
- Outstanding performance.
- Rich standard library.
- Strong tooling.
- Excellent Docker support.

#### Alternatives Considered

- Rust
- Java
- Node.js

#### Trade-offs

Requires writing more infrastructure code compared to full-stack frameworks.

### B.2. HTTP Layer

#### HTTP Server: Go Standard Library `net/http`

The standard library HTTP server is used to serve HTTP requests.

#### Rationale

- Zero external dependency.
- Full control over server lifecycle, timeouts, and graceful shutdown.
- Production-proven at scale.
- Excellent documentation and community support.

#### Alternatives Considered

- FastHTTP
- Fiber

#### Trade-offs

Requires manual configuration of server timeouts and graceful shutdown mechanics.

#### Routing Strategy: Go Standard Library `net/http` (Go 1.22+ Method Patterns)

Go 1.22 introduced method-based routing patterns directly in `net/http` (e.g., `http.HandleFunc("GET /api/v1/urls", handler)`). This eliminates the need for a third-party router while keeping the routing explicit and readable.

#### Rationale

- Zero external dependency.
- Full control over HTTP.
- Idiomatic Go.
- Explicit method + path matching.
- Easier debugging.

#### Alternatives Considered

- chi
- Gin
- Echo

#### Trade-offs

Requires implementing middleware composition manually. Path parameter extraction is more verbose compared to third-party routers.

### B.3. Database - PostgreSQL

#### Driver: `pgx/v5`

#### Rationale

- Native PostgreSQL driver.
- Excellent performance.
- Actively maintained.
- Recommended by the Go PostgreSQL community.

#### Alternatives Considered

- database/sql
- lib/pq

#### Query Layer: `sqlc`

#### Rationale

- SQL-first development.
- Compile-time query validation.
- Type-safe generated code.
- No ORM abstraction.
- Predictable performance.

#### Alternatives Considered

- GORM
- SQLBoiler
- Ent
- Squirrel

#### Trade-offs

Requires writing SQL manually.

#### Migration: `golang-migrate`

#### Rationale

- Mature and battle-tested.
- SQL-first workflow.
- Simple migration model.
- Widely adopted by the Go community.
- Easy integration with CI/CD.

#### Alternatives Considered

- Atlas
- Goose

#### Trade-offs

Does not provide schema diff generation.

### B.4. Cache

#### Technology: Redis

#### Client Library: `go-redis/v9`

#### Rationale

- Officially recommended.
- Mature.
- High performance.
- Excellent documentation.

#### Alternatives Considered

- Memcached
- Dragonfly
- In-memory (local) cache

#### Trade-offs

Introduces an external infrastructure dependency. Requires operational overhead for deployment and monitoring.

#### Cache Strategy Overview

- **Primary use case**: Caching resolved short URL → long URL mappings to reduce database reads on the redirect hot path.
- **Invalidation**: TTL-based expiration with explicit invalidation on URL deletion or update.
- **Consistency model**: Eventual consistency is acceptable; a stale redirect is safe because URLs are immutable once created and updates are rare.

### B.5. Configuration

#### Config: `koanf`

#### Rationale

- Lightweight.
- Flexible.
- Supports multiple configuration sources.
- Environment-first configuration.

#### Alternatives Considered

- Viper
- envconfig

#### Trade-offs

Requires slightly more manual setup.

### B.6. Logging

#### Go Standard Library: `log/slog`

#### Rationale

- Standard library.
- Structured logging.
- Zero external dependency.
- Excellent ecosystem support.

#### Alternatives Considered

- Zap
- Zerolog
- Logrus

#### Trade-offs

Slightly fewer advanced features compared to Zap.

### B.7. Validation

#### Decision

go-playground/validator/v10

#### Rationale

- De facto validation library.
- Rich validation rules.
- Mature ecosystem.

#### Alternatives Considered

- Custom validation

#### Trade-offs

Introduces one additional dependency.

### B.8. Observability

#### Logging: `log/slog`

#### Profiling: Go Standard Library - `net/http/pprof`

#### Metrics: OpenTelemetry Metrics

#### Tracing: OpenTelemetry Tracing

#### Rationale

- Built into Go.
- Production-proven.
- No external dependency.

### B.9. Testing

#### Testing Framework: Go Standard Library (`testing`)

#### Rationale

- Idiomatic Go.
- Zero dependency.
- Excellent integration with Go tooling.

#### Assertion Strategy: Manual assertions using the standard library

Assertions are written as explicit `if` checks with clear failure messages. No third-party assertion library is used.

#### Rationale

- Explicit.
- No additional dependency.
- Encourages understanding of Go testing.
- Easy to read and maintain.

#### Mocking: Manual implementations when necessary

#### Rationale

- Avoid premature abstraction.
- Keep tests easy to understand.
- Introduce mock generators only if future complexity justifies them.

#### Integration Testing: Go Standard Library (`testing`) & Docker Compose

Integration tests spin up real PostgreSQL and Redis instances via Docker Compose to validate end-to-end behavior against actual infrastructure.

#### Rationale

- Tests run against real infrastructure, not mocks.
- Catches integration issues early.
- Docker Compose provides reproducible environments.

#### Benchmarking: `go test -bench`

Go's built-in benchmarking framework is used for micro-benchmarks on critical paths (URL resolution, short code generation, query performance).

#### Rationale

- Zero dependency.
- Built into the Go toolchain.
- Excellent for detecting performance regressions.

#### Load Testing: k6

k6 is used to validate system behavior under expected production traffic volumes.

#### Rationale

- Production-grade.
- Scriptable with JavaScript.
- Supports CI integration.
- Supports realistic workloads.

#### Stress Testing: k6

k6 is also used for stress testing to determine system breaking points and recovery behavior.

#### Rationale

- Same tooling as load testing (shared scripts and CI pipeline).
- Helps identify failure modes and capacity limits.

### B.10. Development

#### Formatter: `gofmt` & `goimports`

#### Linter: `golangci-lint`

#### Security: `gosec` & `govulncheck`

#### Task Runner: Makefile

### B.11. Documentation

#### API Specification: OpenAPI 3.1

### B.12. Containerization

#### Container Runtime: Docker

Docker is used to build reproducible container images for the backend service.

#### Rationale

- Industry standard.
- Excellent Go support with multi-stage builds for minimal image sizes.
- Widely supported by cloud providers and orchestration tools.

#### Alternatives Considered

- Podman

#### Local Development Environment: Docker Compose

Docker Compose orchestrates all local dependencies (PostgreSQL, Redis) for development.

#### Rationale

- Single command to start the full development environment.
- Consistent environment across developers.
- Shared configuration between local development and CI.

### B.13. CI/CD

#### CI Platform: GitHub Actions

#### Rationale

- Native GitHub integration.
- Free for public repositories.
- Extensive marketplace for reusable workflows.
- Docker support built-in.

#### Quality Gates

The CI pipeline enforces the following quality gates on every pull request:

1. **Formatting** — `gofmt` and `goimports` check.
2. **Linting** — `golangci-lint` with project-specific configuration.
3. **Security scanning** — `gosec` for static analysis, `govulncheck` for dependency vulnerabilities.
4. **Unit tests** — `go test ./...` with race detector.
5. **Integration tests** — Tests against PostgreSQL and Redis via Docker Compose.
6. **Build verification** — Successful compilation of the binary.

## C. Guiding Principles

The project follows these technology selection principles.

1. Standard Library First
2. SQL First
3. Explicit Over Magic
4. Production Proven
5. Minimal Dependencies
6. Performance Aware
7. Simplicity Over Cleverness
