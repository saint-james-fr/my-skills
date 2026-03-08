---
name: system-design-api-design
description: API design patterns — REST, GraphQL, gRPC, WebSocket comparison, rate limiting algorithms, API gateway, pagination, idempotency, versioning, and authentication methods. Use when designing APIs, choosing between protocols, implementing rate limiting, setting up API gateways, or evaluating API authentication strategies.
---

# API Design — System Design Reference

## 1. API Protocol Comparison

| Protocol | Data Format | Transport | Use Case | Pros | Cons |
|----------|-------------|-----------|----------|------|------|
| **REST** | JSON/XML | HTTP/1.1+ | CRUD APIs, public APIs, web/mobile backends | Simple, cacheable, widely adopted, stateless | Over/under-fetching, multiple round-trips for related data |
| **GraphQL** | JSON | HTTP | Complex frontends, aggregating multiple sources, mobile apps with bandwidth constraints | Single endpoint, client specifies exact fields, strong typing, subscriptions | Client-side complexity, harder caching, abusive queries possible, N+1 problem |
| **gRPC** | Protobuf (binary) | HTTP/2 | Microservice-to-microservice, low-latency internal APIs, streaming | ~5× faster than JSON, bi-directional streaming, code generation, strong contracts | Not browser-native, binary not human-readable, steeper learning curve |
| **WebSocket** | Any (text/binary) | TCP (upgraded from HTTP) | Real-time apps: chat, live feeds, gaming, collaborative editing | Full-duplex, persistent connection, low overhead per message | Stateful connection, harder to scale/load-balance, no built-in request/response |
| **Webhooks** | JSON (typically) | HTTP POST callback | Event notifications: payment status, CI/CD, third-party integrations | Push-based (real-time), efficient resource usage, simple to implement | Missed notifications if endpoint down, requires retry/idempotency, security concerns |
| **SOAP** | XML | HTTP/SMTP | Enterprise/legacy systems, financial services, WS-Security | Built-in WS-Security, ACID transactions, formal contracts (WSDL) | Verbose XML, heavyweight, complex, poor performance vs REST/gRPC |

### Decision Quick Guide

| Scenario | Pick |
|----------|------|
| Public API consumed by third parties | REST |
| Complex frontend with many nested resources | GraphQL |
| High-throughput internal microservices | gRPC |
| Real-time bidirectional data | WebSocket |
| Notify external systems on events | Webhooks |
| Enterprise with strict security/transaction needs | SOAP (legacy) |

## 2. REST API Design

### HTTP Methods & Idempotency

| Method | Action | Idempotent | Safe | Request Body |
|--------|--------|------------|------|-------------|
| `GET` | Read resource | Yes | Yes | No |
| `POST` | Create resource | **No** | No | Yes |
| `PUT` | Full replace | Yes | No | Yes |
| `PATCH` | Partial update | **No** | No | Yes |
| `DELETE` | Remove resource | Yes | No | No |
| `HEAD` | Get headers only | Yes | Yes | No |
| `OPTIONS` | Get allowed methods | Yes | Yes | No |

### Status Codes Quick Reference

| Range | Meaning | Key Codes |
|-------|---------|-----------|
| **2xx** | Success | `200 OK`, `201 Created`, `202 Accepted`, `204 No Content` |
| **3xx** | Redirection | `301 Moved Permanently`, `304 Not Modified` |
| **4xx** | Client Error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests` |
| **5xx** | Server Error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

### URL Naming Conventions

```
✅ Good                          ❌ Bad
GET /users                       GET /getUsers
GET /users/123                   GET /user?id=123
GET /users/123/orders            GET /getUserOrders?userId=123
POST /users                      POST /createUser
PUT /users/123                   POST /updateUser
DELETE /users/123/orders/456     POST /deleteOrder
```

Rules:
- Use **nouns** (not verbs) for resources
- Use **plural** names (`/users` not `/user`)
- Use **kebab-case** for multi-word paths (`/order-items`)
- Nest sub-resources: `/users/{id}/orders`
- Keep nesting ≤ 3 levels deep

### Versioning Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL path** | `/v1/users` | Simple, explicit, cacheable | URL pollution, harder to sunset |
| **Header** | `Accept: application/vnd.api.v2+json` | Clean URLs, content negotiation | Less discoverable, harder to test |
| **Query param** | `/users?version=2` | Easy to add | Easy to miss, cache key complexity |

Recommendation: URL path versioning is the most common and easiest to maintain.

## 3. Rate Limiting

### Algorithm Comparison

| Algorithm | How It Works | Pros | Cons |
|-----------|-------------|------|------|
| **Token Bucket** | Bucket holds tokens refilled at fixed rate. Each request consumes one token. Request rejected when bucket empty. | Memory efficient, allows bursts, simple | Hard to tune bucket size + refill rate |
| **Leaking Bucket** | FIFO queue processes requests at fixed rate. New requests added to queue; dropped if full. | Smooth output rate, memory efficient | No burst support, old requests block new ones |
| **Fixed Window Counter** | Divide time into fixed windows, count requests per window. Reject when count > threshold. | Simple, memory efficient, resets naturally | Edge-of-window burst problem (2× allowed at boundary) |
| **Sliding Window Log** | Store timestamp of each request in sorted set. Remove outdated entries. Count entries to decide. | Accurate, no boundary issues | High memory (stores every timestamp) |
| **Sliding Window Counter** | Hybrid: weighted sum of current + previous window counters. Formula: `current_count + prev_count × overlap%` | Smooth, memory efficient, accurate (~99.997%) | Approximation, assumes even distribution in previous window |

Used by: Token Bucket → Amazon, Stripe. Leaking Bucket → Shopify. Sliding Window Counter → Cloudflare.

### Where to Place the Rate Limiter

| Location | When to Use |
|----------|------------|
| **Client-side** | Unreliable — client can be forged. Avoid for enforcement. |
| **Server-side middleware** | Full control of algorithm. Good if building your own. |
| **API Gateway** | Most common in microservices. Managed service (AWS API Gateway, Kong, etc.). Handles auth, rate limiting, SSL termination together. |

### Rate Limiter HTTP Headers

| Header | Purpose |
|--------|---------|
| `X-Ratelimit-Remaining` | Remaining allowed requests in current window |
| `X-Ratelimit-Limit` | Total calls allowed per time window |
| `X-Ratelimit-Retry-After` | Seconds to wait before retrying |

When limit is exceeded → return **HTTP 429 Too Many Requests** + `Retry-After` header.

### Distributed Rate Limiting

| Challenge | Solution |
|-----------|---------|
| **Race condition** | Two threads read counter=3, both write 4 (should be 5). Fix: Redis Lua scripts (atomic read+increment), or Redis sorted sets. |
| **Synchronization** | Multiple rate limiter servers need shared state. Fix: centralized Redis store (not sticky sessions). |
| **Multi-region latency** | Edge servers far from central store. Fix: deploy rate limiters at edge + eventual consistency model. |

### Rate Limiting Rules Example (Lyft format)

```yaml
domain: messaging
descriptors:
  - key: message_type
    value: marketing
    rate_limit:
      unit: day
      requests_per_unit: 5

domain: auth
descriptors:
  - key: auth_type
    value: login
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

## 4. API Gateway

### What It Provides

| Function | Description |
|----------|-------------|
| **Request Routing** | Directs API requests to the appropriate backend service |
| **Authentication/Authorization** | Validates identity and permissions before forwarding |
| **Rate Limiting & Throttling** | Controls request volume per client/IP |
| **Load Balancing** | Distributes requests across service instances |
| **Caching** | Stores responses to reduce backend load |
| **Protocol Translation** | REST ↔ gRPC, HTTP ↔ WebSocket |
| **API Composition** | Combines multiple backend calls into single response |
| **SSL Termination** | Handles TLS at the edge |
| **IP Whitelisting** | Restricts access by IP address |
| **Monitoring/Logging** | Centralized request metrics and tracing |

### Reverse Proxy vs API Gateway vs Load Balancer

| Component | Primary Role | Layer | Key Features |
|-----------|-------------|-------|-------------|
| **Reverse Proxy** | Hides backend servers, caching, SSL termination | L7 | Security shield, static content serving, compression |
| **API Gateway** | API management, routing, composition | L7 | Auth, rate limiting, protocol translation, request aggregation |
| **Load Balancer** | Traffic distribution across servers | L4/L7 | Health checks, failover, session persistence, horizontal scaling |

An API gateway often *includes* reverse proxy + load balancer functionality. Use all three when you need fine-grained separation of concerns at scale.

### When to Use API Gateway vs Direct Client-to-Service

| Use API Gateway | Skip API Gateway |
|----------------|-----------------|
| Microservices architecture | Monolith with single backend |
| Multiple client types (web, mobile, IoT) | Single client type, simple routing |
| Need cross-cutting concerns (auth, rate limit) | Auth handled at service level |
| Protocol translation needed | Uniform protocol throughout |
| Public-facing API | Internal-only services |

## 5. Pagination

| Technique | How It Works | Example | Pros | Cons |
|-----------|-------------|---------|------|------|
| **Offset-based** | Skip N rows, return M | `GET /items?offset=20&limit=10` | Simple, supports random access | Slow at large offsets (scans skipped rows), inconsistent if data changes |
| **Cursor-based** | Encode position as opaque token | `GET /items?cursor=eyJpZCI6MTAw&limit=10` | Consistent, efficient for large datasets | No random page access, slightly complex |
| **Keyset-based** | Filter by indexed column value | `GET /items?after_id=100&limit=10` | Very efficient (uses index), stable results | Requires unique indexed key, forward-only |
| **Page-based** | Specify page number + size | `GET /items?page=3&size=10` | Easy to understand, good for UIs | Same perf issues as offset at large pages |
| **Time-based** | Filter by timestamp range | `GET /items?start=2024-01-01&end=2024-01-31` | Natural for chronological data | Requires reliable timestamps |

### Decision Guide

| Scenario | Best Technique |
|----------|---------------|
| Simple admin panel, small dataset | Offset or page-based |
| Infinite scroll, social feed | Cursor-based |
| Large dataset, ordered by ID | Keyset-based |
| Log/event retrieval | Time-based |

## 6. Idempotency

### HTTP Method Idempotency

| Method | Idempotent? | Notes |
|--------|------------|-------|
| `GET` | Yes | Reading doesn't change state |
| `PUT` | Yes | Same full replacement → same result |
| `DELETE` | Yes | Deleting already-deleted resource → same result (404 or 204) |
| `PATCH` | No | Partial updates can compound (e.g., `increment`) |
| `POST` | No | Creates new resource each time |

### Idempotency Key Pattern

For non-idempotent operations (`POST`, `PATCH`), use an **idempotency key**:

```
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "amount": 100,
  "currency": "USD"
}
```

Flow:
1. Client generates unique key (UUID) per logical operation
2. Server checks if key has been seen before
3. If seen → return cached response (no re-execution)
4. If new → process request, store result keyed by idempotency key
5. Key expires after TTL (e.g., 24h)

### Top 6 Cases to Apply Idempotency

| Case | Why |
|------|-----|
| **1. RESTful API requests** | Retried requests must not duplicate operations |
| **2. Payment processing** | Network retries must not double-charge |
| **3. Order management** | Re-submitted orders must not create duplicates |
| **4. Database operations** | Reapplied transactions must not corrupt state |
| **5. User account management** | Registration retries must not create duplicate accounts |
| **6. Distributed messaging** | Reprocessed queue messages must not produce duplicate side effects |

## 7. API Authentication

| Method | How It Works | Security | Use Case |
|--------|-------------|----------|----------|
| **Basic Auth** | Base64-encoded `username:password` in `Authorization` header | Low (credentials in every request) | Internal tools, dev/test environments |
| **API Key** | Static key in header or query param | Medium (key can leak, no user context) | Public APIs, server-to-server, third-party integrations |
| **Bearer Token** | Token in `Authorization: Bearer <token>` | Medium-High (tokens expire) | Session-based auth, mobile apps |
| **OAuth 2.0** | Authorization framework with grants, tokens, refresh tokens | High (scoped access, token rotation) | Third-party access, social login, delegated authorization |
| **JWT** | Self-contained signed token (header.payload.signature) | High (stateless, verifiable, expirable) | Microservices auth, SSO, API-to-API |
| **mTLS** | Mutual TLS with client certificates | Very High | Service mesh, zero-trust, financial APIs |

### OAuth 2.0 Flows Quick Reference

| Flow | When to Use |
|------|------------|
| **Authorization Code** | Web apps with backend (most common) |
| **Authorization Code + PKCE** | Mobile/SPA (no client secret) |
| **Client Credentials** | Machine-to-machine, no user context |
| **Resource Owner Password** | Trusted first-party apps only (legacy) |
| **Implicit** | Deprecated — use Authorization Code + PKCE instead |

### JWT Structure

```
Header.Payload.Signature

Header:  { "alg": "RS256", "typ": "JWT" }
Payload: { "sub": "user123", "exp": 1700000000, "role": "admin" }
Signature: RS256(base64(header) + "." + base64(payload), privateKey)
```

Verify with public key. No server-side session needed. Include `exp` (expiration), `iss` (issuer), `aud` (audience) claims.

### Decision Guide

| Scenario | Recommended |
|----------|-------------|
| Quick prototype / internal tool | API Key or Basic Auth |
| Public API for third parties | OAuth 2.0 + API Key |
| Microservices internal auth | JWT + mTLS |
| Mobile app | OAuth 2.0 Authorization Code + PKCE |
| Third-party delegated access | OAuth 2.0 |
| SSO across multiple services | JWT + OIDC |

## 8. API Performance Tips

| Technique | What It Does |
|-----------|-------------|
| **Pagination** | Return data in pages — don't send entire dataset |
| **Compression** | Use gzip/brotli (`Accept-Encoding: gzip`) to reduce payload size |
| **Connection pooling** | Reuse HTTP/TCP connections — avoid handshake overhead per request |
| **Caching** | Cache at CDN, API gateway, and application level; use `ETag`/`Cache-Control` headers |
| **Async processing** | Return `202 Accepted` for long tasks, process in background, notify via webhook/polling |
| **Partial responses** | Let clients request specific fields (`?fields=id,name,email`) |
| **Batch endpoints** | Allow multiple operations in a single request to reduce round-trips |
| **Rate limiting** | Protect backend from overload and abuse |
| **Database optimization** | Proper indexes, query optimization, read replicas |
| **CDN for static assets** | Serve static responses from edge locations |

### Response Envelope Pattern

```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6MTAwfQ==",
    "has_more": true
  },
  "meta": {
    "request_id": "req_abc123",
    "rate_limit": {
      "remaining": 95,
      "limit": 100,
      "reset": 1700000060
    }
  }
}
```

## Detailed Reference

For in-depth implementations (rate limiter pseudocode, distributed race condition solutions, GraphQL schema patterns, gRPC protobuf examples, API gateway patterns), see [api-design-detail.md](references/api-design-detail.md).
