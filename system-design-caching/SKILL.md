---
name: system-design-caching
description: Caching strategies and patterns — cache tiers, read/write policies, eviction algorithms, CDN, Redis patterns, and cache failure scenarios (thundering herd, penetration, breakdown). Use when designing caching layers, choosing eviction policies, implementing Redis caching, setting up CDN, or debugging cache-related issues.
---

# Caching — System Design Reference

Based on *System Design Interview* (Vol. 1 & 2) by Alex Xu & Sahn Lam; *ByteByteGo* archive.

## 1. Cache Tiers

Data is cached at every layer from client to database.

| Tier | Where | What It Caches | Example |
|------|-------|---------------|---------|
| **Client / Browser** | End-user device | HTTP responses, assets, DNS | `Cache-Control`, `ETag`, Service Worker |
| **CDN** | Edge PoPs worldwide | Static assets (JS, CSS, images, video) | CloudFront, Akamai, Cloudflare |
| **Load Balancer** | Between client and app servers | TLS sessions, HTTP responses | Nginx microcaching |
| **Application** | In-process or sidecar | Computed results, session data, config | In-memory map, Guava Cache |
| **Distributed Cache** | Dedicated cache cluster | Hot data, session state, counters | Redis, Memcached |
| **Full-text Search** | Search engine cluster | Indexed documents, query results | Elasticsearch, OpenSearch |
| **Database** | DB engine internals | Query results, pages, indexes | Buffer pool, WAL, materialized views |
| **CPU / OS** | Hardware + kernel | Instructions, memory pages | L1/L2/L3 cache, page cache |

**Latency reference:**

| Operation | Time |
|-----------|------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| Redis GET (network) | ~0.5 ms |
| SSD random read | ~0.1 ms |
| HDD seek | ~10 ms |

## 2. Read/Write Strategies

| Strategy | How It Works | Pros | Cons | Best For |
|----------|-------------|------|------|----------|
| **Cache-aside** (lazy loading) | App checks cache → miss → read DB → populate cache | Only requested data cached; cache failure non-fatal | Cache miss = 3 round trips; stale data possible | General-purpose reads; tolerates stale data |
| **Read-through** | Cache sits in front of DB; cache itself loads on miss | App code simpler; consistent read path | Cache library must support DB connector; cold start | Read-heavy workloads; uniform access |
| **Write-through** | Write to cache and DB synchronously (cache first) | Cache always consistent with DB; read-after-write safe | Higher write latency (2 writes); caches data that may never be read | Data integrity critical; read-after-write needed |
| **Write-behind** (write-back) | Write to cache → async batch flush to DB | Low write latency; reduces DB write load | Risk of data loss if cache crashes before flush | Write-heavy; burst writes; analytics ingestion |
| **Write-around** | Write directly to DB, skip cache | Avoids polluting cache with write-once data | Cache miss on recently written data | Write-once/rarely-read data; log-style writes |

**Common combinations:** Cache-aside + Write-around (reads are cached on demand, writes go to DB). Read-through + Write-through (full cache layer abstraction).

## 3. Eviction Policies

| Policy | Evicts | When To Use | Trade-off |
|--------|--------|------------|-----------|
| **LRU** (Least Recently Used) | Item not accessed for longest time | General-purpose; temporal locality | Extra bookkeeping (doubly linked list + hash map) |
| **LFU** (Least Frequently Used) | Item with lowest access count | Frequency matters more than recency (CDN assets) | Counter maintenance; slow to adapt to changing patterns |
| **FIFO** (First In First Out) | Oldest item regardless of access | Simple; access pattern doesn't matter | May evict hot items |
| **TTL** (Time-to-Live) | Item whose time-to-live expired | Data has known staleness window | Not strictly an eviction policy; used alongside others |
| **Random** | Random item | Uniform access distribution; simplicity needed | Unpredictable; may evict hot items |
| **MRU** (Most Recently Used) | Most recently accessed item | Scanning workloads where old items are re-accessed | Counterintuitive; narrow use case |
| **SLRU** (Segmented LRU) | Probationary segment items first; protected promoted on re-access | Scan-resistant; separates one-hit wonders from hot data | More complex; two segments to manage |

**Default choice:** LRU covers 80% of use cases. Use LFU for CDN/static assets. Use TTL as a safety net alongside any eviction policy.

## 4. Cache Failure Patterns

| Failure | What Happens | Cause | Prevention |
|---------|-------------|-------|------------|
| **Thundering herd** (cache stampede) | Many keys expire simultaneously → massive DB load spike | Identical TTLs across many keys | Add random jitter to TTLs (`TTL + rand(0, 60s)`); rate-limit non-core DB queries during recovery |
| **Cache penetration** | Queries for non-existent keys bypass cache every time → DB hammered | Malicious or erroneous queries for data that doesn't exist in DB | Cache null results with short TTL; use **Bloom filter** to pre-check key existence |
| **Cache breakdown** (hot key expiry) | Single hot key expires → flood of concurrent requests to DB | Popular key reaches TTL; 80/20 traffic distribution | Never expire hot keys; use mutex/singleflight to let one request rebuild and others wait |
| **Cache crash** | Entire cache layer down → all traffic falls through to DB | Node failure, OOM, network partition | Circuit breaker (fail-fast, don't cascade to DB); deploy cache as cluster across AZs; overprovision memory |

**Mutex / singleflight pattern for breakdown:**

```
GET key → miss
  SETNX key:lock → acquired?
    YES → fetch from DB → SET key → DEL key:lock
    NO  → wait + retry GET key
```

## 5. Redis Caching Patterns

### Data Types for Caching

| Type | Use Case | Example |
|------|----------|---------|
| **String** | Simple key-value; counters; serialized objects | `SET user:123 "{json}"` |
| **Hash** | Object fields; partial updates without full serialization | `HSET user:123 name "Max" age 30` |
| **Sorted Set** | Leaderboards; rate-limiting sliding windows; priority queues | `ZADD leaderboard 9500 user:42` |
| **Set** | Tag membership; unique visitors; intersections | `SADD online_users user:42` |
| **List** | Recent items; activity feeds; queues | `LPUSH recent:user:123 item:456` |
| **HyperLogLog** | Approximate unique counts (12 KB per counter) | `PFADD daily_uv:2024-01-15 user:42` |

### Cache vs Data Store

| Concern | Redis as Cache | Redis as Data Store |
|---------|---------------|-------------------|
| Data loss tolerance | Acceptable (rebuild from DB) | Not acceptable |
| Persistence | Disabled or RDB snapshots | AOF + RDB; `fsync` every write |
| Eviction policy | `allkeys-lru` or `volatile-lru` | `noeviction` |
| Replication | Nice to have | Required (replicas + Sentinel) |
| Cluster | Optional | Required for scale |

### Redis Architecture Evolution

| Year | Version | Milestone |
|------|---------|-----------|
| 2010 | 1.0 | Standalone in-memory cache |
| 2013 | 2.8 | Persistence (RDB snapshots + AOF); replication; Sentinel for monitoring + auto-failover |
| 2015 | 3.0 | Cluster mode — data sharded across 16384 hash slots |
| 2017 | 5.0 | Stream data type |
| 2020 | 6.0 | Multi-threaded network I/O (main processing still single-threaded) |

### Cache Key Design

| Practice | Example | Why |
|----------|---------|-----|
| Prefix with entity type | `user:123:profile` | Avoids collisions; enables key scanning |
| Use colon separators | `geo:hash:9q8yyk` | Convention; readable in monitoring |
| Include version | `v2:user:123:profile` | Safe schema migration; gradual rollout |
| Keep keys short | `u:123:p` (in high-volume cases) | Less memory overhead |
| Avoid user input in keys | Hash or validate input | Prevents injection / unbounded keyspace |

## 6. CDN

### Push vs Pull

| Model | How It Works | Pros | Cons | Best For |
|-------|-------------|------|------|----------|
| **Pull (origin pull)** | CDN fetches from origin on cache miss; caches with TTL | Zero upload management; auto-populates | First request is slow (cold miss); origin must handle spikes on expiry | Most web apps; dynamic content edge caching |
| **Push (origin push)** | You upload content to CDN ahead of time | Full control over what's cached; no origin hit | Must manage upload pipeline; storage cost for unused content | Large static assets; video; known content libraries |

### Cache-Control Headers

| Header | Effect | Example |
|--------|--------|---------|
| `Cache-Control: max-age=N` | Cache for N seconds | `max-age=31536000` (1 year for hashed assets) |
| `Cache-Control: s-maxage=N` | Override max-age for shared caches (CDN) | `s-maxage=3600` (CDN caches 1h) |
| `Cache-Control: no-cache` | Must revalidate with origin every time | Frequently changing HTML pages |
| `Cache-Control: no-store` | Never cache | Auth tokens, private data |
| `Cache-Control: stale-while-revalidate=N` | Serve stale while fetching fresh in background | `stale-while-revalidate=60` |
| `ETag` / `If-None-Match` | Conditional request; 304 if unchanged | Content-hash based validation |
| `Vary` | Cache separately per header value | `Vary: Accept-Encoding` |

### CDN Invalidation Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **TTL expiry** | Set appropriate `max-age`; let content expire naturally | Simple; stale window equals TTL |
| **API purge** | Call CDN vendor API to invalidate specific URLs/paths | Immediate; costs per-purge; rate limits |
| **Versioned URLs** | Append hash or version to filename (`app.a1b2c3.js`) | Instant "invalidation" via new URL; requires build pipeline |
| **Surrogate keys / tags** | Tag cached objects; purge by tag | Purge groups of related content; vendor-specific |

### CDN Considerations

- **Cost**: Charged per GB transferred; avoid caching infrequently accessed assets
- **Fallback**: Clients should detect CDN failure and fall back to origin
- **Multi-CDN**: Use DNS-based routing to multiple CDN providers for resilience and latency optimization
- **Dynamic content**: Edge compute (CloudFront Functions, Cloudflare Workers) can cache personalized content at the edge

## 7. Cache Consistency

### Invalidation Approaches

| Approach | Description | Consistency | Complexity |
|----------|------------|-------------|------------|
| **TTL-based** | Cache expires after fixed time | Eventual (bounded staleness) | Low |
| **Event-driven invalidation** | DB change → publish event → invalidate cache | Near real-time | Medium |
| **Write-through** | Update cache and DB in same operation | Strong (within single region) | Medium |
| **Double-delete** | Delete cache → update DB → delay → delete cache again | Handles race conditions | Medium |
| **Change Data Capture** | Stream DB binlog (Debezium) → invalidate/update cache | Near real-time; decoupled | High |

### DB-Cache Consistency Patterns

**Problem:** Update DB and cache are two separate operations → race conditions.

| Pattern | Steps | Guarantees |
|---------|-------|------------|
| **Cache-aside invalidation** | Update DB → delete cache key (not update) | Next read rebuilds from DB; avoids stale writes |
| **Delete then update (risky)** | Delete cache → update DB | Race: concurrent read can re-cache old value |
| **Double-delete** | Delete cache → update DB → sleep(N) → delete cache again | Clears stale value from concurrent read race |
| **Write-through + read-through** | All reads/writes go through cache layer | Strong consistency if single writer; serialization needed |

**Rule of thumb:** Delete cache keys on write, don't update them. Updates create ordering races; deletes are idempotent.

### TTL Tuning Guidelines

| Data Type | Suggested TTL | Rationale |
|-----------|--------------|-----------|
| Static assets (CSS, JS, images) | Long (1 year) + versioned URLs | Content-addressed; new version = new URL |
| User profile / preferences | 5–15 min | Changes infrequently; tolerate short staleness |
| Product catalog / pricing | 1–5 min | Balance freshness vs DB load |
| Session data | Match session timeout | Security requirement |
| Rate limit counters | Window size (1 min / 1 hour) | Must expire to reset window |
| Real-time data (stock prices) | No cache or < 1s | Staleness unacceptable |
| Search results | 30s–5 min | Acceptable staleness; high compute cost to rebuild |

## 8. Decision Guide

### When to Cache

| Cache When | Don't Cache When |
|-----------|-----------------|
| Read-heavy workload (read:write > 10:1) | Write-heavy workload with strong consistency needs |
| Expensive computation to reproduce | Data changes every request |
| Data changes infrequently | Data is security-sensitive and must not be stored in volatile memory |
| High latency to data source | Dataset is tiny and already in-memory (no benefit) |
| Data is tolerant of bounded staleness | Cache management complexity exceeds latency benefit |

### Cache Sizing

| Factor | Guidance |
|--------|----------|
| **Working set** | Cache the hot 20% that serves 80% of reads |
| **Memory budget** | Overprovision by 25–30% for safety margin and spikes |
| **Key count × avg size** | Estimate: `num_keys × (key_size + value_size + 64 bytes overhead)` |
| **Eviction rate** | Monitor; high eviction = cache too small or TTL too short |
| **Hit ratio target** | Aim for > 95%; below 80% reassess strategy or sizing |

### Quick Sizing Example

```
200M business records × 8 bytes (ID) × 3 precisions = ~5 GB
→ Fits in a single modern Redis instance
→ Deploy cluster across regions for HA and latency
```

### SPOF Avoidance

- Never rely on a single cache server
- Deploy across multiple availability zones
- Overprovision memory by 25–30%
- Implement circuit breaker: if cache is down, fail fast rather than cascade to DB

## Detailed Reference

For Netflix caching case study, Redis architecture deep dive, distributed cache consistency patterns, and CDN optimization, see [caching-detail.md](references/caching-detail.md).
