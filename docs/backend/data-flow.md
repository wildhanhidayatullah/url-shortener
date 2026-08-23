# Data Flow

## A. Overview

This document describes how data flows through the system for each business use case.

The purpose is to document the interaction between architectural components instead of implementation details.

All requests generally follow the same high-level architecture:

```
                Client
                   │
                   ▼
             HTTP Transport
                   │
                   ▼
            Application Layer
                   │
                   ▼
              Domain Layer
                   │
                   ▼
          Infrastructure Layer
                   │
                   ▼
         External Dependencies
```

Business rules are implemented inside the Domain layer. Infrastructure is responsible only for technical concerns.

## B. Cross-Cutting Request Flow

Every HTTP request follows the same processing pipeline.

```
                Client
                   │
                   ▼
              HTTP Request
                   │
                   ▼
          Recovery Middleware
                   │
                   ▼
          Request ID Middleware
                   │
                   ▼
            Structured Logging
                   │
                   ▼
              Rate Limiting
                   │
                   ▼
              HTTP Handler
                   │
                   ▼
              Application
                   │
                   ▼
                 Domain
                   │
                   ▼
             Infrastructure
                   │
                   ▼
              HTTP Response
                   │
                   ▼
            Response Logging
```

Business-specific processing begins inside the Application layer.

### B.1. Create Short URL

```
Client
    │
    ▼
POST /api/v1/urls
    │
    ▼
Validate Request
    │
    ▼
Validate Business Rules
    │
    ▼
Generate Short Code
(or validate custom alias)
    │
    ▼
Check Alias Availability
    │
    ▼
Persist URL
    │
    ▼
Return Created URL
```

If a custom alias already exists, the request fails with **409 Conflict**.

### B.2. Update URL

```
Client
    │
    ▼
PUT /api/v1/urls/{id}
    │
    ▼
Validate Request
    │
    ▼
Find URL
    │
    ▼
Validate Business Rules
    │
    ▼
Update URL
    │
    ▼
Invalidate Cache
    │
    ▼
Return Updated Resource
```

### B.3. Soft Delete URL

```
Client
    │
    ▼
DELETE /api/v1/urls/{id}
    │
    ▼
Find URL
    │
    ▼
Mark as Deleted
    │
    ▼
Invalidate Cache
    │
    ▼
Return No Content
```

URLs are never physically removed.

### B.4. Redirect Flow

Redirect latency is the highest priority of the system.

Click analytics must never delay redirects.

```
Client
    │
    ▼
GET /{shortCode}
    │
    ▼
Lookup Redis Cache
    │
    ├── Cache Hit ──────────────────────────┐
    │                                       │
    ▼                                       │
Cache Miss                                  │
    │                                       │
    ▼                                       │
Lookup PostgreSQL                           │
    │                                       │
    ├── Not Found ──► Return 404            │
    │                                       │
    ▼                                       │
Validate Deleted                            │
    │                                       │
    ▼                                       │
Validate Expiration                         │
    │                                       │
    ▼                                       │
Populate Cache                              │
    │                                       │
    └───────────────────────────────────────┘
                    │
                    ▼
          Extract Click Metadata
                    │
                    ▼
              Validate Bot
                    │
                    ├── Bot ──► Skip Analytics, Redirect
                    │
                    ▼
           Publish Click Event
                    │
                    ▼
              302 Redirect
```

Cache hits bypass PostgreSQL entirely. Bot traffic is detected during metadata validation; bots are redirected but do not generate analytics events. The redirect response never waits for analytics persistence.

### B.5. Analytics Processing

Analytics processing is asynchronous.

The redirect request publishes an event and immediately returns the redirect response.

```
Redirect Request
        │
        ▼
Create Click Event
        │
        ▼
Buffered Channel
        │
        ├──────────────┐
        ▼              ▼
    Worker #1      Worker #2
        │              │
        └──────┬───────┘
               ▼
        Persist Analytics
               │
               ▼
          PostgreSQL
```

The analytics worker is responsible only for persistence.

Metadata extraction is performed before the event enters the queue.

### B.6. Dashboard Flow

```
Client
    │
    ▼
GET /api/v1/dashboard
    │
    ▼
Validate Request
    │
    ▼
Read URL List
    │
    ▼
For Each URL:
    │
    ├─► Total Click Count
    ├─► Clicks Today
    ├─► Clicks This Week
    ├─► Clicks This Month
    ├─► Top Browsers
    ├─► Top Referrers
    └─► Device Distribution
    │
    ▼
Aggregate Results
    │
    ▼
Return JSON Response
```

All analytics queries are read-only operations against PostgreSQL. The dashboard aggregates click data across multiple dimensions (time-based, browser, referrer, device) in a single request cycle. Future caching may be introduced without changing the business flow.

### B.7. Graceful Shutdown

The application performs graceful shutdown to prevent losing queued analytics events whenever possible.

```
SIGTERM
    │
    ▼
Stop Accepting New Requests
    │
    ▼
Wait for Active Requests
    │
    ▼
Drain Analytics Queue
    │
    ▼
Workers Finish Processing
    │
    ▼
Shutdown
```

## C. Failure Scenarios

### Invalid Request

Return: `400 Bad Request`

- Malformed JSON body
- Missing required fields
- Invalid URL format
- Invalid custom alias format

### Short URL Not Found

Return: `404 Not Found`

### URL Expired

Return: `410 Gone`

### URL Soft Deleted

Return: `404 Not Found`

Deleted URLs are treated as non-existent.

### Redis Unavailable

- Fallback to PostgreSQL.
- The request continues normally.

### Analytics Queue Full

- The analytics event is dropped.
- The redirect request continues normally.
- Redirect latency has higher priority than analytics completeness.

### Analytics Persistence Failure

- The event is logged.
- The request is not retried.
- The redirect response is unaffected.

### PostgreSQL Unavailable

Return: `503 Service Unavailable`

Create, Update, Delete, Redirect, and Dashboard operations fail.

## D. Architectural Decisions

### Redirect Performance First

Redirect requests must never wait for analytics persistence.

### Asynchronous Analytics

Analytics processing uses an in-memory buffered channel and worker goroutines.

### Best-Effort Analytics

Analytics collection prioritizes system responsiveness over event durability. Events may be dropped when the queue is full or persistence fails.

### Cache-Aside Strategy

Redis is used as a cache. PostgreSQL remains the source of truth.

### Graceful Degradation

Failure of analytics processing must not prevent URL redirection. Failure of Redis must not prevent system operation. The system should continue operating with reduced functionality whenever possible.

## E. Data Persistence

### Storage Responsibilities

| Store      | Role            | Data                                          |
| ---------- | --------------- | --------------------------------------------- |
| PostgreSQL | Source of truth | URL records, analytics events, click metadata |
| Redis      | Cache           | Short URL resolution cache (short_code → URL) |
| In-Memory  | Transient queue | Analytics events awaiting persistence         |

### PostgreSQL

PostgreSQL is the primary data store. All URL records and analytics events are persisted here.

**URL Records** — short_code, original_url, expires_at, deleted_at, timestamps.

**Analytics Events** — click_id, url_id, click_time, referrer, user_agent, browser, os, device_type, ip_hash, country.

### Redis

Redis serves as a read-through cache for the redirect hot path.

- **Key pattern**: short code as key, original URL as value.
- **TTL**: Configurable expiration.
- **Invalidation**: Explicit invalidation on URL update or delete.
- **Consistency**: Eventual consistency is acceptable; stale redirects are safe.

### Read/Write Flow

```
Write Path (URL Management):
    Application → Domain Validation → PostgreSQL

Read Path — Cache Hit (Redirect):
    Application → Redis → Domain → 302 Redirect

Read Path — Cache Miss (Redirect):
    Application → Redis (miss) → PostgreSQL → Domain → Redis (populate) → 302 Redirect

Write Path (Analytics):
    Redirect Handler → Buffered Channel → Worker → PostgreSQL

Read Path (Dashboard):
    Application → PostgreSQL (aggregation) → Response
```

### Data Persistence Boundaries

- **Application layer** decides what to persist and when.
- **Domain layer** enforces business rules but does not perform I/O.
- **Infrastructure layer** executes actual database and cache operations.
- The Application layer never directly interacts with PostgreSQL or Redis; it calls repository interfaces defined at the Domain or Application boundary.
