# API Design — Detailed Reference

## 1. Rate Limiting Algorithm Implementations

### Token Bucket

```
function tokenBucket(request):
    bucket = getBucket(request.clientId)
    now = currentTime()

    // Refill tokens based on elapsed time
    elapsed = now - bucket.lastRefill
    bucket.tokens = min(
        bucket.maxTokens,
        bucket.tokens + elapsed * bucket.refillRate
    )
    bucket.lastRefill = now

    if bucket.tokens >= 1:
        bucket.tokens -= 1
        saveBucket(bucket)
        return ALLOW
    else:
        return REJECT
```

Parameters:
- `maxTokens` (bucket capacity): controls burst size
- `refillRate` (tokens/second): controls sustained throughput

Example: `maxTokens=10, refillRate=2/s` → allows burst of 10, sustained 2 req/s.

Used by: Amazon (API Gateway), Stripe.

### Leaking Bucket

```
function leakyBucket(request):
    queue = getQueue(request.clientId)

    if queue.size >= maxQueueSize:
        return REJECT

    queue.enqueue(request)
    return ALLOW

// Separate process drains the queue at fixed rate
function drainLoop():
    while true:
        sleep(1 / outflowRate)
        if queue.isNotEmpty():
            request = queue.dequeue()
            processRequest(request)
```

Parameters:
- `maxQueueSize`: how many waiting requests to buffer
- `outflowRate`: requests processed per second

Key difference from token bucket: output rate is strictly constant (no bursts).

Used by: Shopify.

### Fixed Window Counter

```
function fixedWindowCounter(request):
    windowKey = floor(currentTime() / windowSize)
    key = request.clientId + ":" + windowKey

    count = redis.INCR(key)
    if count == 1:
        redis.EXPIRE(key, windowSize)

    if count > maxRequests:
        return REJECT
    return ALLOW
```

Edge problem: 5 requests at 2:00:59 + 5 at 2:01:01 = 10 in a 2-second span, despite a limit of 5/minute. The boundary between windows creates a vulnerability.

### Sliding Window Log

```
function slidingWindowLog(request):
    now = currentTime()
    windowStart = now - windowSize
    key = request.clientId

    // Remove expired entries
    redis.ZREMRANGEBYSCORE(key, 0, windowStart)

    // Count current entries
    count = redis.ZCARD(key)

    if count >= maxRequests:
        return REJECT

    // Add current request timestamp
    redis.ZADD(key, now, request.id)
    redis.EXPIRE(key, windowSize)
    return ALLOW
```

Redis sorted set stores timestamps. Accurate but memory-heavy (one entry per request).

### Sliding Window Counter

```
function slidingWindowCounter(request):
    now = currentTime()
    currentWindow = floor(now / windowSize)
    prevWindow = currentWindow - 1
    positionInWindow = (now % windowSize) / windowSize

    currentCount = redis.GET(clientId + ":" + currentWindow) or 0
    prevCount = redis.GET(clientId + ":" + prevWindow) or 0

    // Weighted estimate
    estimate = currentCount + prevCount * (1 - positionInWindow)

    if estimate >= maxRequests:
        return REJECT

    redis.INCR(clientId + ":" + currentWindow)
    return ALLOW
```

Only stores 2 counters per client. Cloudflare reports 0.003% error rate across 400M requests.

## 2. Distributed Rate Limiter Design

### Architecture

```
Client → Load Balancer → Rate Limiter Middleware → API Server
                              ↓
                         Redis Cluster
                         (centralized counters)
```

### Race Condition Problem

Two concurrent requests read counter=3, both increment to 4. Expected: 5.

```
Thread A: read(counter) → 3
Thread B: read(counter) → 3
Thread A: write(counter, 4)
Thread B: write(counter, 4)  // Lost update!
```

### Solution: Redis Lua Script (Atomic Operations)

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = tonumber(redis.call('GET', key) or '0')

if current >= limit then
    return 0  -- rejected
end

if current == 0 then
    redis.call('SET', key, 1, 'EX', window)
else
    redis.call('INCR', key)
end

return 1  -- allowed
```

Lua scripts execute atomically in Redis — no race conditions. The entire read-check-increment sequence is a single operation.

### Solution: Redis Sorted Sets

```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]

-- Remove expired entries
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current entries
local count = redis.call('ZCARD', key)

if count >= limit then
    return 0
end

redis.call('ZADD', key, now, member)
redis.call('EXPIRE', key, window)
return 1
```

### Synchronization Across Multiple Rate Limiters

Problem: Client 1 → Rate Limiter A, Client 2 → Rate Limiter B. Without shared state, each limiter has partial view.

Solutions:
- **Centralized Redis**: all rate limiters read/write to the same Redis cluster
- **Sticky sessions**: route same client to same limiter (not recommended — reduces flexibility)
- **Eventual consistency**: for multi-region setups, allow brief over-limit at edges, sync asynchronously

### Multi-Data Center Setup

Deploy rate limiter instances at edge locations. Traffic routed to closest edge server. Synchronize counters with eventual consistency to tolerate cross-region latency.

```
User (US-West) → Edge Rate Limiter (US-West) → Redis (US-West)
                                                    ↕ async sync
User (EU)      → Edge Rate Limiter (EU)      → Redis (EU)
```

### Monitoring Checklist

- Is the algorithm dropping valid requests? → relax rules
- Is a traffic burst overwhelming the system? → switch to token bucket
- Are rate limit rules too permissive? → tighten thresholds
- Hard vs soft limiting: hard = strict cutoff, soft = allow brief overages

## 3. GraphQL Schema Design Patterns

### Basic Schema Structure

```graphql
type Query {
  user(id: ID!): User
  users(filter: UserFilter, first: Int, after: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
}

type Subscription {
  userCreated: User!
  orderStatusChanged(orderId: ID!): Order!
}
```

### Input/Payload Pattern

Wrap mutation inputs and outputs for forward compatibility:

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: UserRole
}

type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UserError {
  field: String!
  message: String!
  code: ErrorCode!
}
```

### Connection Pattern (Relay-style Pagination)

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

Usage: `query { users(first: 10, after: "cursor123") { edges { node { name } } pageInfo { hasNextPage } } }`

### Avoiding N+1 Queries

```graphql
# This query fetches 10 users, then 10 separate queries for orders
query {
  users(first: 10) {
    edges {
      node {
        name
        orders { id total }  # N+1 problem here
      }
    }
  }
}
```

Solution: **DataLoader** — batches and deduplicates database calls within a single request.

```typescript
const orderLoader = new DataLoader(async (userIds: string[]) => {
  const orders = await db.orders.findByUserIds(userIds)
  return userIds.map(id => orders.filter(o => o.userId === id))
})

// Resolver
const resolvers = {
  User: {
    orders: (user) => orderLoader.load(user.id)
  }
}
```

### Query Complexity & Depth Limiting

Prevent abusive queries:

```typescript
const validationRules = [
  depthLimit(5),
  costAnalysis({
    maximumCost: 1000,
    defaultCost: 1,
    variables: context.variables,
    onComplete: (cost: number) => {
      if (cost > 1000) throw new Error('Query too expensive')
    }
  })
]
```

## 4. gRPC Protobuf Examples

### Service Definition (.proto)

```protobuf
syntax = "proto3";
package order.v1;

service OrderService {
  // Unary RPC
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);

  // Server streaming: server sends multiple responses
  rpc ListOrders(ListOrdersRequest) returns (stream Order);

  // Client streaming: client sends multiple requests
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (BatchCreateResponse);

  // Bidirectional streaming
  rpc OrderUpdates(stream OrderUpdateRequest) returns (stream OrderUpdateResponse);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  string idempotency_key = 3;
}

message Order {
  string id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  google.protobuf.Timestamp created_at = 5;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 price_cents = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

### gRPC Communication Patterns

| Pattern | Client | Server | Use Case |
|---------|--------|--------|----------|
| **Unary** | 1 request | 1 response | Standard request/response (CRUD) |
| **Server streaming** | 1 request | N responses | Real-time feeds, large result sets |
| **Client streaming** | N requests | 1 response | File upload, telemetry data |
| **Bidirectional streaming** | N requests | N responses | Chat, live collaboration |

### gRPC Data Flow

```
Client App
  → gRPC Stub (generated code)
    → Serialize to Protobuf (binary)
      → HTTP/2 transport
        → Server receives packets
          → Deserialize from Protobuf
            → Server App processes
              → Serialize response to Protobuf
                → HTTP/2 back to client
                  → Deserialize
                    → Client receives typed response
```

### gRPC vs REST Comparison

| Aspect | gRPC | REST |
|--------|------|------|
| Payload | Protobuf (binary, ~5× smaller) | JSON (text) |
| Transport | HTTP/2 (multiplexed) | HTTP/1.1 or HTTP/2 |
| Contract | Strict `.proto` file | OpenAPI/Swagger (optional) |
| Code generation | Built-in (polyglot) | Third-party tools |
| Streaming | Native (4 patterns) | Workarounds (SSE, WebSocket) |
| Browser support | Requires gRPC-Web proxy | Native |
| Caching | Not built-in | HTTP caching (ETag, Cache-Control) |
| Debugging | Binary, needs tools | Human-readable JSON |

## 5. API Gateway Patterns

### Backend for Frontend (BFF)

Each client type gets its own API gateway tailored to its needs:

```
Mobile App  → Mobile BFF Gateway  → Microservices
Web App     → Web BFF Gateway     → Microservices
IoT Device  → IoT BFF Gateway     → Microservices
```

Each BFF:
- Aggregates only the data its client needs
- Handles device-specific concerns (mobile: smaller payloads, offline; web: SSR)
- Owned by the frontend team for that platform

### Aggregation Gateway

Single gateway combines multiple backend calls:

```
Client: GET /dashboard

Gateway internally calls:
  → GET /users/123         (User Service)
  → GET /users/123/orders  (Order Service)
  → GET /users/123/stats   (Analytics Service)

Gateway returns combined response:
{
  "user": { ... },
  "recentOrders": [ ... ],
  "stats": { ... }
}
```

Reduces client round-trips from 3 to 1.

### Gateway Offloading

Move cross-cutting concerns out of services:

| Concern | Without Gateway | With Gateway |
|---------|----------------|-------------|
| Auth | Every service validates tokens | Gateway validates once |
| Rate limiting | Each service implements its own | Gateway enforces globally |
| SSL | Each service manages certs | Gateway terminates SSL |
| Logging | Each service logs requests | Gateway logs uniformly |
| CORS | Each service sets headers | Gateway handles globally |

### Gateway Routing Patterns

```yaml
routes:
  - path: /api/v1/users/**
    service: user-service
    strip_prefix: /api/v1

  - path: /api/v1/orders/**
    service: order-service
    strip_prefix: /api/v1

  - path: /api/v1/payments/**
    service: payment-service
    strip_prefix: /api/v1
    rate_limit:
      requests_per_second: 10
      burst: 20

  - path: /api/v1/search/**
    service: search-service
    timeout: 5s
    retry:
      attempts: 3
      backoff: exponential
```

### Popular API Gateway Solutions

| Gateway | Type | Best For |
|---------|------|----------|
| **Kong** | Open source | Full-featured, plugin ecosystem |
| **AWS API Gateway** | Managed | AWS-native, serverless |
| **Nginx** | Open source | High-performance reverse proxy + gateway |
| **Envoy** | Open source | Service mesh sidecar, gRPC-native |
| **Traefik** | Open source | Container-native, auto-discovery |
| **Apigee** | Managed (Google) | Enterprise API management |

## 6. WebSocket Design Considerations

### Connection Lifecycle

```
Client                          Server
  |---- HTTP Upgrade Request ----->|
  |<--- 101 Switching Protocols ---|
  |                                |
  |<====== WebSocket Frames ======>|  (full-duplex)
  |                                |
  |---- Close Frame -------------->|
  |<--- Close Frame Ack ----------|
```

### Scaling WebSocket Servers

| Challenge | Solution |
|-----------|---------|
| Stateful connections | Sticky sessions at load balancer (by connection ID) |
| Server memory per connection | Limit connections per server, horizontal scale |
| Cross-server messaging | Pub/sub (Redis Pub/Sub, Kafka) for broadcasting |
| Reconnection storms | Exponential backoff with jitter on client reconnect |
| Health checks | Application-level ping/pong frames |

### When WebSocket vs SSE vs Polling

| Technique | Direction | Use Case |
|-----------|-----------|----------|
| **WebSocket** | Bidirectional | Chat, gaming, collaborative editing |
| **SSE (Server-Sent Events)** | Server → Client only | Live dashboards, notifications, stock tickers |
| **Long Polling** | Simulated push | Legacy support, simple notifications |
| **Short Polling** | Client → Server | Low-frequency checks, status polling |

## 7. API Versioning Deep Dive

### Breaking vs Non-Breaking Changes

| Breaking (requires new version) | Non-Breaking (safe to add) |
|--------------------------------|---------------------------|
| Remove a field | Add optional field |
| Rename a field | Add new endpoint |
| Change field type | Add new query parameter |
| Change response structure | Add new response field |
| Remove an endpoint | Add new enum value |
| Change authentication method | Deprecate (but keep) endpoint |

### Sunset Strategy

```http
Sunset: Sat, 01 Mar 2025 00:00:00 GMT
Deprecation: true
Link: <https://api.example.com/v3/docs>; rel="successor-version"
```

Recommended timeline:
1. Announce deprecation (6+ months before removal)
2. Add `Deprecation` and `Sunset` headers
3. Log usage of deprecated endpoints
4. Notify consumers directly
5. Remove after sunset date
