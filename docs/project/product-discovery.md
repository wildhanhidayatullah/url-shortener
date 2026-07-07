# Product Discovery

## A. Goals

Convert long URLs into short URLs while collecting click analytics with minimal redirect latency.

## B. Features

### URL

1. Create a short URL
2. Redirect to the original URL
3. Custom alias
4. Expiration date
5. Soft delete
6. Restore soft-deleted URL

### Analytics

1. Click time
2. Referrer
3. User agent
4. Browser
5. Operating system
6. Device type
7. IP address (stored as a hash or anonymized to protect privacy)
8. Country (using GeoIP)
9. Ignore bot traffic

### Dashboard

1. List of URLs
2. Number of clicks
3. Click analytics
4. Clicks today
5. Clicks this week
6. Clicks this month
7. Top browsers
8. Top referrers
9. Device distribution

## C. Functional Requirements

### URL Management

1. The system can create a short URL
2. The system can update the destination URL
3. The system can delete the URL
4. The system can create a short URL with custom alias
5. The system can return URL details
6. The system can set the expiration time for URLs
7. The system can unset the expiration time for URLs

### Redirect

1. The system can find the right URL by short code
2. The system can validate the soft deleted URL
3. The system can validate the expiration time
4. The system can publish a click analytics event
5. The system can redirect users

### Analytics

1. The system can ignore bot traffics
2. The system publishes an analytics event for every valid click, example:

```
Time: 2026-07-02T10:30:21Z
Referrer: google.com
User Agent: Mozilla/5.0 ...
Browser: Chrome
OS: Windows
Device: Desktop
IP Address Hashed: ...
Country: Indonesia
```

## D. Non-Functional Requirements

### Infrastructure

| Area             | Requirement         |
| ---------------- | ------------------- |
| Database         | Relational Database |
| Cache            | Required            |
| Containerization | Required            |
| CI/CD            | Required            |

### Observability

| Area    | Requirement        |
| ------- | ------------------ |
| Logging | Structured Logging |
| Metrics | Required           |
| Tracing | Required           |

### Development

| Area              | Requirement                     |
| ----------------- | ------------------------------- |
| Configuration     | Environment-based               |
| Health Check      | Required                        |
| Graceful Shutdown | Required                        |
| Testing           | Unit, Integration, Load, Stress |

### Security

| Area                     | Target              |
| ------------------------ | ------------------- |
| Input Validation         | Required            |
| SQL Injection Protection | Parameterized Query |
| Rate Limiting            | Yes                 |
| CORS                     | Configurable        |
| Security Headers         | Configurable        |

### Performance

| Area              | Target  |
| ----------------- | ------- |
| Throughput        | 1000TPS |
| Redirect Latency  | < 100ms |
| Dashboard Latency | < 500ms |

## E.Success Metrics

| Metric                | Target                      |
| --------------------- | --------------------------- |
| Redirect Success Rate | > 97%                       |
| Redirect Latency      | < 100 ms                    |
| Analytics Event Loss  | < 1% under normal operation |
| Availability          | > 97%                       |

## F. Assumptions & Constraints

- Single region deployment
- Single PostgreSQL instance
- Single Redis instance
- No authentication
- No custom domain
- REST API only
- Asynchronous in-memory analytics processing
- Redis is used as a cache, PostgreSQL remains the source of truth
