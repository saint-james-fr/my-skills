# Caching — Detailed Reference

## Netflix Caching: 4 Ways Netflix Uses Caching

Netflix is one of the most cache-intensive architectures in the industry. Their approach covers four distinct layers.

### 1. EVCache — Distributed Caching Layer

Netflix built EVCache (Ephemeral Volatile Cache) on top of Memcached, optimized for AWS:

| Feature | Detail |
|---------|--------|
| **Replication** | Data replicated across multiple AZs for resilience |
| **Client-side sharding** | Consistent hashing distributes keys across nodes |
| **Warm-up** | New nodes are pre-populated from replicas before serving traffic |
| **Scale** | Trillions of requests/day; tens of millions of operations/second |
| **Fallback** | On miss → fetch from data store → populate cache asynchronously |

**Use cases at Netflix:**
- User session and profile data
- Viewing history and recommendations
- Authentication tokens
- API response caching

### 2. CDN — Open Connect

Netflix operates its own CDN called **Open Connect** rather than relying on third-party CDNs for video:

| Aspect | Detail |
|--------|--------|
| **Architecture** | Custom hardware appliances (OCAs) deployed inside ISP networks |
| **Content** | Pre-positioned video content during off-peak hours (push model) |
| **Routing** | Steering service directs clients to optimal OCA based on network conditions |
| **Scale** | Serves 100% of Netflix video traffic globally |
| **Fallback** | Multi-OCA redundancy; clients can switch to alternative OCA mid-stream |

**Why not a traditional CDN?** Video represents ~95% of Netflix bandwidth. Owning the CDN allows hardware optimization, ISP-level placement, and cost control at scale.

### 3. Application-Level Caching

Netflix services cache data in-process using Guava/Caffeine caches:

- **Metadata service**: caches movie/show metadata with TTL-based invalidation
- **API aggregation layer**: caches assembled responses for device-specific formats
- **Fallback data**: stale cached data served during upstream failures (graceful degradation)

### 4. Database Caching

- **CockroachDB / MySQL**: buffer pool caching of hot pages
- **Cassandra**: key cache + row cache for frequently read partitions
- **Read replicas**: serve as a "cache" for read-heavy workloads

### Lessons from Netflix

1. **Cache at every layer** — don't rely on a single cache tier
2. **Own your CDN** at scale — third-party CDN costs become prohibitive at Netflix's volume
3. **Graceful degradation** — serve stale data rather than errors
4. **Warm-up is critical** — cold cache on new nodes causes cascading failures

## Redis Architecture Deep Dive

### Evolution Timeline

| Year | Redis Version | Key Change | Impact |
|------|--------------|------------|--------|
| 2010 | 1.0 | Standalone in-memory store | Simple cache; all data lost on restart |
| 2013 | 2.8 | RDB snapshots + AOF persistence | Data survives restarts |
| 2013 | 2.8 | Master-replica replication | Read scaling; basic HA |
| 2013 | 2.8 | Sentinel | Automatic failover; monitoring |
| 2015 | 3.0 | Cluster mode | Horizontal scaling via 16384 hash slots |
| 2017 | 5.0 | Streams data type | Event sourcing; message queue use cases |
| 2020 | 6.0 | Multi-threaded I/O | Network I/O parallelism; main loop still single-threaded |
| 2022 | 7.0 | Redis Functions; multi-part AOF | Server-side scripting; faster persistence |

### Persistence Deep Dive

| Feature | RDB (Snapshot) | AOF (Append-Only File) |
|---------|---------------|----------------------|
| **How** | Point-in-time binary snapshot via `fork()` | Every write command appended to log |
| **Durability** | Data loss between snapshots | Configurable: every write, every second, or OS flush |
| **Recovery speed** | Fast (load binary) | Slower (replay commands); AOF rewrite compacts |
| **CPU impact** | Spike during `fork()` + child write | Minimal per-write; rewrite is background |
| **Recommendation** | Use for backups; fast recovery | Use for durability; combine with RDB for best of both |

### Sentinel vs Cluster

| Feature | Sentinel | Cluster |
|---------|----------|---------|
| **Purpose** | High availability (failover) | HA + horizontal scaling |
| **Data model** | Full dataset on every node | Data sharded across nodes (16384 hash slots) |
| **Scaling** | Vertical only (bigger instance) | Horizontal (add nodes, reshard) |
| **Multi-key operations** | All keys on same instance | Only within same hash slot (`{tag}` prefix) |
| **Complexity** | Lower | Higher |
| **When to use** | Dataset fits in single instance; need HA | Dataset exceeds single instance; need throughput scaling |

### Redis Memory Optimization

| Technique | Effect |
|-----------|--------|
| Use Hashes for small objects (< 100 fields) | ziplist encoding uses ~10x less memory than individual keys |
| Short key names in high-volume scenarios | Saves bytes per key × millions of keys |
| Set `maxmemory-policy` explicitly | Prevents OOM; controls eviction behavior |
| Use `OBJECT ENCODING key` to inspect | Verify Redis is using compact encodings |
| Avoid storing large values (> 1 MB) | Causes latency spikes; fragment memory |
| `UNLINK` instead of `DEL` for large keys | Non-blocking deletion in background thread |

## Distributed Cache Consistency Patterns

### The Core Problem

Updating a database and a cache are two separate operations. Without transactions spanning both systems, race conditions are inevitable.

### Pattern 1: Cache-Aside with Delete-on-Write

```
Write path:
  1. Update DB
  2. Delete cache key (don't update it)

Read path:
  1. Read cache → hit? return
  2. Miss → read DB → write to cache → return
```

**Why delete instead of update?**
- Two concurrent updates: Update A writes to DB, Update B writes to DB, Update B writes to cache, Update A writes to cache → cache has stale value from A
- Delete is idempotent; the next read always rebuilds from the latest DB state

### Pattern 2: Double-Delete

```
1. Delete cache key
2. Update DB
3. Sleep(estimated_replication_lag)
4. Delete cache key again
```

Handles the race where a concurrent read between steps 1 and 2 re-caches the old DB value. The second delete clears it.

**Limitation:** The sleep duration is a guess. Too short = race still possible. Too long = unnecessary delay.

### Pattern 3: Change Data Capture (CDC)

```
App → writes to DB only (never touches cache directly)
DB binlog → Debezium/Maxwell → message bus → cache updater service → updates/invalidates cache
```

| Advantage | Disadvantage |
|-----------|-------------|
| App code never manages cache | Additional infrastructure (Debezium, Kafka) |
| Single source of truth (DB) | Eventual consistency (replication lag) |
| Works for all write paths (including migrations, backfills) | Ordering guarantees require careful partition design |
| Decoupled; cache logic separate from business logic | Higher operational complexity |

### Pattern 4: Lease-Based Invalidation (Facebook Memcache)

From Facebook's "Scaling Memcache" paper:

```
1. Client reads cache → miss → Memcache issues a lease token
2. Client reads DB → writes to cache with lease token
3. If another invalidation arrived while client was reading DB,
   the lease is revoked → write rejected → stale data prevented
```

This prevents the stale-set problem where a slow reader caches an outdated value after a newer write has already invalidated the key.

### Consistency Spectrum

| Level | Mechanism | Staleness Window | Use Case |
|-------|-----------|-----------------|----------|
| **Strong** | Write-through + read-through with locking | None (within region) | Financial data, inventory counts |
| **Near real-time** | CDC / event-driven invalidation | Milliseconds–seconds | User profiles, product catalog |
| **Eventual (bounded)** | TTL-based expiry | TTL duration | Search results, recommendations |
| **Eventual (unbounded)** | No active invalidation; rely on TTL | Until TTL expires | Static content, rarely-read data |

## CDN Optimization Strategies

### Cache Hit Ratio Optimization

| Strategy | How | Impact |
|----------|-----|--------|
| **Versioned file names** | `app.a3f8d2.js` → infinite `max-age` | 100% hit rate for static assets; instant "invalidation" via new URL |
| **Normalize query strings** | Sort/strip irrelevant params before cache key | Prevents cache fragmentation |
| **Collapse `Vary` headers** | Only vary on what matters (`Accept-Encoding`, not `User-Agent`) | Fewer cache variants |
| **Stale-while-revalidate** | Serve cached content while fetching fresh in background | Higher hit rate; lower perceived latency |
| **Pre-warm popular content** | Push content to CDN before traffic spike (e.g., product launch) | Avoids thundering herd on origin |

### Multi-CDN Architecture

| Component | Role |
|-----------|------|
| **DNS-based routing** | Route users to optimal CDN based on geography/performance |
| **Real-time monitoring** | Measure CDN performance per-region; detect outages |
| **Automatic failover** | Shift traffic from degraded CDN to healthy provider |
| **Origin shield** | Intermediate cache layer between CDN and origin; reduces origin load |

**When to use multi-CDN:**
- Global audience with varying CDN performance by region
- High availability requirements (CDN provider outage)
- Negotiating leverage with CDN vendors
- Compliance requirements (data sovereignty)

### Edge Compute

Modern CDNs offer compute at the edge, enabling dynamic content caching:

| Platform | Runtime | Use Case |
|----------|---------|----------|
| Cloudflare Workers | V8 isolates | Personalization, A/B testing, auth at edge |
| CloudFront Functions | JavaScript | URL rewrites, header manipulation, simple logic |
| Lambda@Edge | Node.js / Python | Origin request/response transformation |
| Fastly Compute | Wasm | High-performance edge logic |

Edge compute enables caching of "personalized" content by computing variations at the edge rather than routing to origin.

### CDN Cost Control

| Technique | Effect |
|-----------|--------|
| Set appropriate `Cache-Control` headers | Reduce origin fetches |
| Compress with Brotli/gzip at origin | Smaller transfer sizes |
| Remove rarely-accessed content from CDN | Avoid paying for storage + transfer of cold content |
| Use origin shield | Collapse multiple CDN-to-origin requests into one |
| Analyze access logs | Identify what's cached vs what's always missing |
| Request-collapsing / coalescing | CDN merges concurrent cache misses into single origin fetch |
