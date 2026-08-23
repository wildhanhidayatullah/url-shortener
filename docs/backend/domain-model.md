# Domain Model

## A. Overview

This document describes the domain model of the URL Shortener service. The project adopts a pragmatic Domain-Driven Design (DDD-lite) approach, where the domain is modeled around business concepts while avoiding unnecessary complexity.

The primary goal is to produce a domain model that is easy to understand, maintain, and evolve as the system grows.

## B. Domain Boundaries

### B.1. URL Domain

Responsible for the lifecycle of shortened URLs. Responsibilities include:

- Creating short URLs
- Updating destination URLs
- Managing expiration
- Managing soft deletion
- Resolving redirects

### B.2. Analytics Domain

Responsible for collecting and providing click analytics. Responsibilities include:

- Recording click events
- Storing analytics data
- Serving analytics queries

Analytics is intentionally separated from the URL domain because it has different responsibilities, data characteristics, and scalability concerns.

### B.3. Domain Model Overview

```
┌─────────────────────────────────────────────────────────┐
│                    URL Domain                           │
│                                                         │
│  ┌─────────────────────────────┐                        │
│  │        URL (Aggregate)      │                        │
│  │  ┌───────────────────────┐  │                        │
│  │  │ ShortCode (VO)        │  │                        │
│  │  │ OriginalURL (VO)      │  │                        │
│  │  │ ExpiredAt             │  │                        │
│  │  │ DeletedAt             │  │                        │
│  │  └───────────────────────┘  │                        │
│  └──────────────┬──────────────┘                        │
│                 │                                       │
└─────────────────┼───────────────────────────────────────┘
                  │ references via URL ID
                  ▼
┌─────────────────────────────────────────────────────────┐
│                 Analytics Domain                        │
│                                                         │
│  ┌─────────────────────────────┐                        │
│  │       Click (Aggregate)     │                        │
│  │  ┌───────────────────────┐  │                        │
│  │  │ HashedIP (VO)         │  │                        │
│  │  │ Timestamp             │  │                        │
│  │  │ Browser, OS, Device   │  │                        │
│  │  │ Country, Referrer     │  │                        │
│  │  └───────────────────────┘  │                        │
│  └─────────────────────────────┘                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## C. Aggregates

### C.1. URL Aggregate

The URL aggregate represents the lifecycle of a shortened URL.

#### Aggregate Root

```
URL
```

#### State

```
ID
ShortCode
OriginalURL
ExpiredAt
DeletedAt
CreatedAt
UpdatedAt
```

#### Responsibilities

- Create URL
- Update destination URL
- Determine redirect eligibility
- Validate expiration
- Validate soft deletion

The URL aggregate is the consistency boundary for all URL-related business rules.

### C.2. Click Aggregate

The Click aggregate represents a single redirect event.

#### Aggregate Root

```
Click
```

#### State

```
ID
URLID
Timestamp
Browser
OperatingSystem
Device
Country
Referrer
HashedIP
```

#### Responsibilities

- Represent a single click event
- Preserve analytics information

Click records are immutable once created.

## D. Aggregate Relationships

```
URL (1)
    │
    │
    └───────────────┐
                    │
                Click (N)
```

A single URL may have many associated click events.

The Click aggregate references the URL only through `URLID`, keeping aggregate boundaries independent.

## E. Entities

The project currently defines two entities.

### E.1. URL

Represents a shortened URL and its lifecycle.

Characteristics:

- Has identity
- Mutable
- Supports soft deletion
- Supports expiration

### E.2. Click

Represents a recorded redirect event.

Characteristics:

- Has identity
- Immutable after creation
- Append-only

## F. Value Objects

The project introduces Value Objects only where they encapsulate meaningful domain rules.

### F.1. ShortCode

Represents a generated or custom short identifier.

Responsibilities may include:

- Length validation
- Character validation
- Reserved word validation

### F.2. OriginalURL

Represents the destination URL.

Responsibilities may include:

- URL validation
- URL normalization
- Scheme validation

### F.3. HashedIP

Represents an anonymized client IP address.

Responsibilities may include:

- Hash format validation
- Preventing storage of raw IP addresses

The following fields remain primitive types because they currently contain no domain-specific behavior:

- Browser
- OperatingSystem
- Device
- Country
- Referrer
- ExpiredAt

The model intentionally avoids introducing unnecessary Value Objects.

## G. Repository Interfaces

Repositories abstract persistence for aggregates.

### G.1. URLRepository

Responsibilities include:

- Save
- Update
- FindByID
- FindByShortCode
- SoftDelete

The repository does not contain business rules.

### G.2. ClickRepository

Responsibilities include:

- Persist click events
- Retrieve analytics data

Because analytics data is append-only, update and delete operations are not required for the MVP.

## H. Domain Services

The current domain model does not introduce Domain Services.

Business rules are sufficiently encapsulated within aggregates and coordinated by the application layer.

Domain Services may be introduced in future iterations if business logic spans multiple aggregates.

## I. Domain Events

The system uses an asynchronous event pipeline for analytics processing.

```
Redirect
    │
    ▼
Create Click Event
    │
    ▼
Buffered Channel
    │
    ▼
Worker Pool
    │
    ▼
PostgreSQL
```

This event flow is considered an application architecture concern rather than a full Domain Event implementation.

The project intentionally avoids introducing explicit Domain Events until the business domain requires broader event-driven behavior.

## J. Business Invariants

The following business rules must always hold.

### J.1. URL

- ShortCode must be unique.
- OriginalURL must be valid.
- Soft-deleted URLs cannot be redirected.
- Expired URLs cannot be redirected.
- A URL is redirect-eligible only when it is neither soft-deleted nor expired.

### J.2. Click

- Every Click must reference a valid URL at creation time.
- Click records are immutable once persisted.

## K. Domain Behavior

The following primary behaviors are associated with the domain model.

### K.1. Create URL

- Accept original URL and optional custom alias
- Generate or validate short code
- Enforce short code uniqueness
- Persist the new URL record

### K.2. Update Destination URL

- Find existing URL by ID
- Update the destination URL
- Invalidate cached resolution

### K.3. Soft Delete URL

- Find existing URL by ID
- Mark URL as soft-deleted
- Invalidate cached resolution
- Prevent future redirects

### K.4. Determine Redirect Eligibility

- Resolve short code to URL record
- Verify URL is not soft-deleted
- Verify URL is not expired
- Reject request if either condition fails

### K.5. Create Click Event

- Extract click metadata from request
- Validate referenced URL exists and is eligible
- Persist click record asynchronously
- Never block the redirect response

## L. Future Evolution

The domain model is intentionally designed for incremental evolution.

Possible future enhancements include:

- User domain
- Workspace domain
- Authentication
- Custom domains
- Restore soft-deleted URL
- Richer analytics metadata
- Explicit Domain Services
- Explicit Domain Events

These additions can be introduced without significant restructuring because aggregate boundaries remain small and well-defined.

## M. Design Trade-offs

| Decision                  | Rationale                                                                   |
| ------------------------- | --------------------------------------------------------------------------- |
| Two aggregates            | Matches the current business complexity.                                    |
| Small aggregates          | Easier to understand, test, and evolve.                                     |
| Minimal Value Objects     | Avoids unnecessary abstraction.                                             |
| No child entities         | The current domain does not require aggregate hierarchies.                  |
| No Domain Services        | Existing business rules remain simple.                                      |
| No explicit Domain Events | The asynchronous analytics pipeline is sufficient for current requirements. |

## N. Summary

The domain model follows the project's engineering principles:

- Domain First
- Pragmatic Clean Architecture
- DDD-lite
- Keep It Simple
- Explicit over Magic
- Avoid Premature Optimization

The model prioritizes clarity, maintainability, and incremental evolution while remaining aligned with the project's current business requirements.
