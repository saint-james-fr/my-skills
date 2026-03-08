# Scaling Fundamentals — Detailed Reference

Companion to `SKILL.md`. Deep dives on sharding, load balancing, CDN, replication, and real-world case studies.

---

## 1. Sharding Algorithms in Depth

### 1.1 Hash-Based Sharding

Simplest approach: `shard = hash(key) % num_shards`.

```
key = "user_42"
hash("user_42") = 283741
shard = 283741 % 4 = 1  →  route to Shard 1
```

**Problem**: Adding a shard (4 → 5) changes the modulus, invalidating nearly all mappings. Most data must be relocated.

### 1.2 Consistent Hashing

Solves the resharding problem. Nodes and keys are mapped onto a hash ring (0 to 2³²-1).

```
         0
        ╱   ╲
      N₁      N₂         N = node
     ╱          ╲         K = key
    K₁           K₂
     ╲          ╱
      N₃      N₄
        ╲   ╱
        2³²
```

- Each key walks clockwise from its position to the first node it encounters.
- Adding node N₅ only affects keys between N₅'s predecessor and N₅.
- On average, only `K/N` keys need to move when a node is added (K = total keys, N = total nodes).

**Virtual nodes**: Each physical node maps to multiple positions on the ring. This:
- Smooths out uneven distribution
- Allows weighted assignment (stronger node gets more virtual nodes)
- Standard practice: 100-200 virtual nodes per physical node

### 1.3 Range-Based Sharding

Partition by contiguous ranges of the shard key.

```
Shard 0: user_id     1 – 1,000,000
Shard 1: user_id 1,000,001 – 2,000,000
Shard 2: user_id 2,000,001 – 3,000,000
```

**Advantages**:
- Range queries within a shard are efficient (e.g. "all users 1-1000")
- Simple to understand and implement

**Disadvantages**:
- Prone to hotspots if access patterns correlate with key ranges
- New users all write to the latest shard (monotonic increase)

**Mitigation**: Use compound shard keys (e.g. `region + user_id`) or randomize key prefix.

### 1.4 Directory-Based Sharding

A lookup service (directory) maps each key to its shard.

```
┌──────────────────┐
│  Lookup Table    │
│  key_range → shard│
│  "US" → Shard 0  │
│  "EU" → Shard 1  │
│  "APAC" → Shard 2│
└────────┬─────────┘
         │ query
         ▼
    ┌─────────┐
    │ Shard N │
    └─────────┘
```

**Advantages**:
- Maximum flexibility — can rebalance without changing hash function
- Supports arbitrary partitioning logic

**Disadvantages**:
- Lookup table is a single point of failure
- Every query requires a directory lookup (latency overhead)
- Directory must be highly available and cached

### 1.5 Virtual Bucket Sharding

Two-level indirection: keys → virtual buckets → physical shards.

```
Key ──hash──▶ Virtual Bucket (0-999) ──mapping──▶ Physical Shard (0-3)

Buckets 0-249   → Shard 0
Buckets 250-499 → Shard 1
Buckets 500-749 → Shard 2
Buckets 750-999 → Shard 3
```

When adding Shard 4, reassign some virtual buckets — only data in those buckets moves.
This gives consistent-hashing-like benefits with simpler implementation.

### Sharding Algorithm Comparison

| Algorithm | Data movement on resize | Distribution evenness | Complexity | Range queries |
|-----------|------------------------|----------------------|------------|---------------|
| Hash mod N | ~100% keys move | Good (with good hash) | Low | Not supported |
| Consistent hashing | ~K/N keys move | Good (with virtual nodes) | Medium | Not supported |
| Range-based | Only new range moves | Depends on key distribution | Low | Supported |
| Directory-based | Configurable | Configurable | High (directory mgmt) | Supported |
| Virtual buckets | Only reassigned buckets | Good | Medium | Not supported |

---

## 2. Load Balancer Configuration Patterns

### 2.1 Health Checks

Load balancers must detect unhealthy backends to avoid routing traffic to dead servers.

| Check type | How it works | When to use |
|-----------|-------------|-------------|
| TCP check | Open TCP connection to port | Basic liveness — can the process accept connections? |
| HTTP check | Send GET to `/health` endpoint | Application-level health (can it serve requests?) |
| Deep health check | `/health` verifies DB, cache, dependencies | Full readiness — all dependencies available |
| Passive check | Monitor response codes from real traffic | Detect degradation without extra probe traffic |

**Health check parameters**:
- `interval`: 5-30s between checks
- `threshold`: 2-3 consecutive failures before marking unhealthy
- `timeout`: 2-5s per check

### 2.2 SSL/TLS Termination Patterns

| Pattern | Where TLS terminates | Backend traffic | Use case |
|---------|---------------------|----------------|----------|
| TLS termination at LB | Load balancer | Plain HTTP | Simplest; offloads crypto from backends |
| TLS pass-through | Backend server | Encrypted end-to-end | Compliance requirements (e.g. PCI DSS) |
| TLS re-encryption | Load balancer decrypts, re-encrypts to backend | Encrypted both legs | Security + LB needs to inspect L7 headers |

### 2.3 Connection Draining

When removing a server from the pool (deploy, scale-down):

1. Stop sending **new** connections to the server
2. Allow **existing** connections to complete (drain period: 30-300s)
3. After drain timeout, forcefully close remaining connections
4. Remove server from pool

### 2.4 Rate Limiting at the LB

| Strategy | Scope | Implementation |
|----------|-------|----------------|
| Per-IP rate limit | Individual client | Token bucket / sliding window at LB |
| Per-endpoint rate limit | API route | L7 LB inspects URL path |
| Global rate limit | Entire service | Distributed counter (Redis-backed) |

### 2.5 Where to Place Load Balancers

```
                     Internet
                        │
                   ┌────▼────┐
                   │  L7 LB  │  ← SSL termination, route by path
                   └────┬────┘
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          ┌──────┐  ┌──────┐  ┌──────┐
          │Web 1 │  │Web 2 │  │Web 3 │
          └──┬───┘  └──┬───┘  └──┬───┘
             └─────────┼─────────┘
                  ┌────▼────┐
                  │  L4 LB  │  ← TCP level, fast
                  └────┬────┘
             ┌─────────┼─────────┐
             ▼         ▼         ▼
         ┌──────┐  ┌──────┐  ┌──────┐
         │App 1 │  │App 2 │  │App 3 │
         └──┬───┘  └──┬───┘  └──┬───┘
            └─────────┼─────────┘
                 ┌────▼────┐
                 │  L4 LB  │  ← DB connection pooling
                 └────┬────┘
            ┌─────────┼─────────┐
            ▼         ▼         ▼
        ┌──────┐  ┌──────┐  ┌──────┐
        │DB Ldr│  │DB F1 │  │DB F2 │
        └──────┘  └──────┘  └──────┘
```

### Reverse Proxy vs API Gateway vs Load Balancer

| Component | Primary job | Extra capabilities |
|-----------|------------|-------------------|
| **Reverse Proxy** | Hide backend topology; forward requests | Caching, compression, SSL termination |
| **API Gateway** | Single entry point for microservices | Auth, rate limiting, request transformation, routing |
| **Load Balancer** | Distribute traffic across identical servers | Health checks, session persistence, failover |

In practice, many tools (Nginx, Envoy, AWS ALB) serve multiple roles.

---

## 3. CDN Caching Strategies

### 3.1 Push vs Pull CDN

| Model | How content gets to edge | Best for |
|-------|-------------------------|----------|
| **Pull (origin-pull)** | CDN fetches from origin on first request; caches until TTL | Most use cases; simple setup |
| **Push (origin-push)** | You upload content to CDN ahead of time | Large files, predictable content (video, software releases) |

### 3.2 Cache-Control Headers

| Header | Effect | Example |
|--------|--------|---------|
| `Cache-Control: max-age=86400` | Cache for 24 hours | Static assets |
| `Cache-Control: s-maxage=3600` | CDN caches for 1 hour (browser may differ) | API responses cached at edge |
| `Cache-Control: no-cache` | Must revalidate with origin | Dynamic content |
| `Cache-Control: no-store` | Never cache | Sensitive data |
| `Surrogate-Control` | CDN-specific cache directive | Fine-grained CDN control |

### 3.3 Invalidation Strategies

| Strategy | How it works | Trade-off |
|----------|-------------|-----------|
| **TTL expiry** | Content expires naturally | Simple; stale window = TTL duration |
| **Purge API** | Call CDN API to invalidate specific URLs | Immediate; but API call overhead |
| **URL versioning** | `app.js?v=abc123` or `app.abc123.js` | No purge needed; immutable URLs; requires build pipeline |
| **Soft purge** | Mark stale but serve until origin responds | Balances freshness and availability |

**Best practice**: Use content-hash-based filenames for static assets (`main.a1b2c3.js`). Serve `index.html` with short TTL or `no-cache`.

### 3.4 Multi-CDN Strategy

Large-scale systems use multiple CDN providers:
- DNS-based routing to pick CDN based on geography, performance, or cost
- Failover: if one CDN is degraded, route to another
- Negotiation: different CDNs for different asset types

---

## 4. Database Replication Topologies

### 4.1 Single-Leader (Leader–Follower)

```
        ┌────────┐
  W ──▶ │ Leader │
        └───┬────┘
       ┌────┼────┐
       ▼    ▼    ▼
      F₁   F₂   F₃  ◀── R
```

- Writes go to leader only
- Followers replicate asynchronously (or semi-synchronously)
- Read replicas scale read throughput linearly

**Replication lag**: Followers may serve stale reads. Mitigations:
- Read-your-writes: after a write, read from leader for N seconds
- Monotonic reads: pin user to one follower within a session
- Semi-synchronous: at least one follower is guaranteed in-sync

### 4.2 Multi-Leader

```
      DC-East              DC-West
    ┌────────┐           ┌────────┐
    │Leader A│◀──sync───▶│Leader B│
    └───┬────┘           └───┬────┘
        ▼                    ▼
    Followers            Followers
```

- Each DC has its own leader accepting writes
- Asynchronous replication between leaders
- **Conflict resolution** required: last-write-wins, merge, custom logic
- Use case: multi-data center, offline-capable clients

### 4.3 Leaderless (Dynamo-style)

```
Client ──W──▶ Node₁, Node₂, Node₃  (write to quorum)
Client ──R──▶ Node₁, Node₃         (read from quorum)
```

- No designated leader — any node accepts writes
- Quorum: `W + R > N` ensures consistency (e.g. W=2, R=2, N=3)
- Anti-entropy and read-repair heal inconsistencies
- Used by: Cassandra, DynamoDB, Riak

### Replication Topology Comparison

| Topology | Write throughput | Read throughput | Consistency | Complexity | Multi-DC |
|----------|-----------------|-----------------|-------------|------------|----------|
| Single-leader | 1x (leader only) | Nx (N followers) | Strong (sync) or eventual (async) | Low | Possible (follower in remote DC) |
| Multi-leader | Nx (N leaders) | Nx | Eventual (conflict resolution) | High | Native |
| Leaderless | Nx | Nx | Tunable (quorum) | High | Possible |

---

## 5. Real-World Case Studies

### 5.1 Netflix: Multi-Data Center Active-Active

**Challenge**: Serve 200M+ subscribers globally with zero downtime.

**Architecture**:
- Active-active across 3 AWS regions (US-East, US-West, EU)
- GeoDNS routes users to nearest region
- Cassandra for data replication (leaderless, eventual consistency)
- EVCache (Memcached-based) for caching — per-region instances
- Zuul as L7 API gateway with failover routing

**Scaling patterns used**:
- Stateless web tier — any instance handles any request
- Async processing via message queues for encoding, recommendations
- CDN (Open Connect) — Netflix's own CDN embedded in ISPs
- Database sharding and denormalization for personalization data

**Failover**: Zuul detects region unhealthy → shifts traffic to other regions within seconds. Cassandra's multi-DC replication ensures data is available.

### 5.2 Pinterest: Database Sharding Journey

**Challenge**: Outgrew single MySQL instance at ~10M users.

**Decisions**:
- Evaluated clustering solutions (MySQL Cluster, Cassandra) — rejected due to complexity
- Chose application-level sharding with MySQL
- Shard key: object ID encodes shard number directly

**ID structure**: `[shard_id (16 bits)][type (10 bits)][local_id (38 bits)]`
- 2¹⁶ = 65,536 possible shards
- 2³⁸ = ~274 billion IDs per shard per type

**Sharding scheme**:
- Each shard is a MySQL database with identical schema
- Application routing layer maps ID → shard → physical DB host
- Multiple shards per physical host (virtual bucket approach)

**Growth strategy**:
- Start with many virtual shards on few hosts
- As data grows, move virtual shards to new hosts
- No resharding — the virtual shard mapping is the only thing that changes

**Lessons**:
- Keep the sharding layer simple — just ID-based routing
- Avoid cross-shard queries — denormalize aggressively
- Use Redis/Memcached heavily to reduce DB load

### 5.3 StackOverflow: Vertical Scaling Success Story

**Challenge**: Serve 10M+ monthly unique visitors.

**Approach**: Contrary to conventional wisdom, StackOverflow ran on a remarkably small footprint:
- 1 primary SQL Server (with 1 replica)
- 11 web servers behind HAProxy
- Redis for caching
- Elasticsearch for search

**Why vertical scaling worked**:
- SQL Server on a beefy machine (384 GB RAM, 1.8 TB SSD)
- Aggressive caching (Redis) reduced DB load to manageable levels
- Well-optimized queries and indexes
- Read-heavy workload (most visits are reads)

**Takeaway**: Don't shard prematurely. Vertical scaling + caching can take you very far. Shard when you have genuine write bottlenecks or data volume that exceeds single-machine capacity.

---

## 6. Capacity Planning Quick Reference

### Back-of-Envelope Numbers

| Component | Throughput | Latency |
|-----------|-----------|---------|
| Single web server (Node.js / Go) | 10K-50K req/s | ~1-10ms |
| Redis (single node) | 100K ops/s | <1ms |
| PostgreSQL (read, indexed) | 10K-50K qps | ~1-5ms |
| PostgreSQL (write) | 1K-10K tps | ~5-20ms |
| Kafka (single broker) | 100K-200K msg/s | ~2-5ms |
| CDN edge | Millions req/s (distributed) | <10ms to user |

### When to Scale What

| Signal | Action |
|--------|--------|
| CPU > 70% sustained on web servers | Add web nodes (horizontal) |
| DB read latency increasing | Add read replicas or cache layer |
| DB write latency increasing | Vertical scale first, then shard |
| Queue depth growing continuously | Add consumer workers |
| P99 latency high for static assets | Add CDN or more edge locations |
| Single-region outage unacceptable | Add data center in another region |
