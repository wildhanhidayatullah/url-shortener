# Architecture

## A. Overview

This project is a backend service that provides URL shortening and click analytics.

The primary goals are:

- Convert long URLs into short URLs.
- Redirect users with minimal latency.
- Record click analytics for every valid request.
- Provide aggregated analytics through REST APIs.

The architecture prioritizes simplicity, maintainability, and clear domain boundaries while remaining easy to evolve as the project grows.

This document is aligned with the Product Discovery defined in `docs/project/product-discovery.md`. Key architectural decisions (PostgreSQL, Redis, structured logging, metrics, tracing, async analytics) are driven by the non-functional requirements and constraints documented there.

## B. Architecture Style

### Decision

The project adopts **Pragmatic Clean Architecture with DDD-lite**.

### Principles

- The domain is independent from frameworks and infrastructure.
- Business rules are implemented inside the domain layer.
- Application use cases orchestrate business workflows.
- Infrastructure implements external dependencies.
- Transport is responsible for protocols (HTTP).
- Dependencies always point toward the domain.

### Trade-offs Considered

| Alternative                                                       | Why Not                                                                                                                         |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Full Hexagonal (Ports & Adapters)                                 | Overhead of ports/adapters terminology for a small service; Clean Architecture covers the same goal with less ceremony.         |
| Traditional Layered Architecture (n-tier)                         | Does not enforce domain independence; risks business logic leaking into transport or infrastructure layers.                     |
| Full DDD (Aggregates, Value Objects, Domain Events, Repositories) | Too heavyweight for the current scope; DDD-lite captures the essential benefit (domain isolation) without the extra complexity. |

The selected style balances clarity, enforceability, and pragmatism for a service with two domains and a single-team development model.

## C. Domain Model

The project is organized around business domains instead of technical layers.

Current domains:

- URL
- Analytics

The Redirect capability belongs to the URL domain because it represents one of the URL lifecycle use cases instead of an independent business capability.

Future domains may include:

- Auth
- API Key
- User
- Administration

### Domain Interaction

The URL domain and Analytics domain interact through the Application layer, not directly:

```
Redirect Use Case (Application)
    │
    ├─► URL Domain ── find & validate short URL
    │
    └─► Analytics Domain ── publish click event (async)
```

- The redirect use case belongs to the URL domain and is responsible for locating and validating the short URL.
- After a successful redirect, the Application layer emits a click event to the Analytics domain.
- Analytics processing is asynchronous (in-memory) to keep the redirect path fast and decoupled.
- The Analytics domain never calls the URL domain; dependency flows one way: URL → Analytics via Application orchestration.

## D. Architectural Components (Layer Responsibilities)

### Transport

Responsible for:

- HTTP routing
- Request parsing
- Response serialization
- Middleware

Transport contains no business logic.

### Application

Responsible for:

- Executing use cases
- Coordinating domain objects
- Managing transactions
- Calling infrastructure through abstractions when necessary

Application contains workflow logic but not business rules.

### Domain

Responsible for:

- Business entities
- Domain services
- Domain validation
- Domain errors
- Business invariants

The domain must not depend on HTTP, PostgreSQL, Redis, or any external library.

### Infrastructure

Responsible for:

- PostgreSQL
- Redis
- Logging
- OpenTelemetry
- Configuration
- External integrations

Infrastructure depends on the domain, never the opposite.

## E. Dependency Rule

Dependencies always point inward.

```
  Transport
      │
      ▼
 Application
      │
      ▼
    Domain
      ▲
      │
Infrastructure
```

The domain never imports infrastructure or transport packages.

### Dependency Inversion

Infrastructure layer implements interfaces defined by the Application or Domain layer (consumer-owned interfaces). This inverts the traditional dependency direction: high-level policy (domain rules) does not depend on low-level detail (database, cache, external APIs); low-level detail depends on abstractions defined by consumers.

## F. Design Principles

### Domain First

Business requirements drive the design.

### Keep It Simple

Avoid unnecessary abstractions.

### Explicit Dependencies

All dependencies are injected explicitly. Global state is avoided.

### Small Interfaces

Interfaces are introduced only when they provide real value. Interfaces belong to the consumer, not the implementation.

### Standard Library First

Prefer Go's standard library whenever practical.

### Context Propagation

Operations that may block accept context.Context.

### Fail Fast

Validate inputs early. Reject invalid requests as soon as possible.

### Security by Default

- Validate all inputs.
- Use parameterized SQL.
- Apply rate limiting.
- Never expose internal errors.

## G. Evolution Strategy

The architecture is intentionally designed for the current scope while allowing future evolution.

Examples:

- Background workers
- Message queues
- Multiple executables
- Authentication
- API Keys
- Horizontal scaling

These capabilities should be added only when justified by new requirements. Premature optimization and unnecessary abstractions should be avoided.
