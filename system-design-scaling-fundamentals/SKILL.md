---
name: system-design-scaling-fundamentals
description: Core scaling patterns from single server to millions of users — load balancing, CDN, database replication and sharding, stateless architecture, and message queues. Use when designing scalable systems, choosing between vertical and horizontal scaling, deciding on data partitioning strategies, or setting up a high-level architecture.
---

# Scaling Fundamentals

## 1. Scaling Evolution Path

Each step addresses a bottleneck exposed by the previous stage.

| Stage | What changes | Bottleneck solved |
|-------|-------------|-------------------|
| 1. Single server | Web + DB + cache on one box | — |
| 2. Separate DB | Dedicated database server | Web & DB compete for CPU/RAM |
| 3. Load balancer + multiple web servers | LB in front of ≥2 web nodes | Single web server = SPOF, no horizontal scale |
| 4. DB replication (leader → follower) | Reads go to followers | DB is single point of failure; read bottleneck |
| 5. Cache layer (Redis / Memcached) | Hot data served from memory | Repeated DB reads for same data |
| 6. CDN | Static assets served from edge | Latency for geographically distributed users |
| 7. Stateless web tier | Session state externalized to data store | Can't auto-scale web nodes; sticky sessions |
| 8. Multiple data centers | Geo-routed traffic, replicated data | Regional outage takes down entire service |
| 9. Message queues | Async processing, decoupled producers/consumers | Tight coupling; spiky workloads overwhelm sync processing |
| 10. Database sharding | Horizontal data partitioning | Single DB can't hold all data; write bottleneck |

```
Stage 1        Stage 3              Stage 10 (full)

┌──────┐      ┌────┐               ┌────┐
│Client│      │ LB │               │ LB │
└──┬───┘      └─┬──┘               └─┬──┘
   │            ├────┐            ┌──┴──┐
┌──▼───┐    ┌──▼──┐ ┌▼────┐   ┌──▼──┐ ┌▼────┐
│Server │    │Web 1│ │Web 2│   │Web 1│ │Web N│  (stateless)
│(all)  │    └──┬──┘ └─┬───┘   └──┬──┘ └─┬───┘
└───────┘       └──┬───┘          └──┬───┘
               ┌───▼───┐        ┌───▼───┐   ┌───────┐
               │  DB   │        │Cache  │   │  MQ   │
               └───────┘        └───┬───┘   └───┬───┘
                                ┌───▼───┐   ┌───▼───┐
                                │Leader │   │Workers│
                                └───┬───┘   └───────┘
                               ┌────┼────┐
                            ┌──▼┐ ┌▼──┐ ┌▼──┐
                            │F 1│ │F 2│ │F N│ (followers)
                            └───┘ └───┘ └───┘
                            ┌────────────────┐
                            │  Shard 0..N    │
                            └────────────────┘
```

## 2. Vertical vs Horizontal Scaling

| Dimension | Vertical (scale up) | Horizontal (scale out) |
|-----------|-------------------|----------------------|
| How | Add CPU / RAM / disk to existing machine | Add more machines to the pool |
| Simplicity | Simple — no code changes | Requires load balancing, stateless design |
| Hard limit | Yes — hardware ceiling | No — add nodes indefinitely |
| Failover | None — single machine | Built-in — other nodes absorb traffic |
| Cost curve | Superlinear (bigger boxes cost disproportionately more) | Near-linear |
| Downtime | Often requires restart | Zero-downtime rolling deploys |
| Best for | Small/medium workloads, early stage, DB tier (initially) | Large-scale, web tier, stateless services |

**Rule of thumb**: Start vertical (simple), switch to horizontal when you hit limits or need redundancy. DB tier often stays vertical longer than web tier.

## 3. Load Balancing

### L4 vs L7 Comparison

| Aspect | L4 (Transport) | L7 (Application) |
|--------|----------------|-------------------|
| OSI layer | TCP/UDP | HTTP/HTTPS/WebSocket |
| Routing decision | IP + port | URL path, headers, cookies, body |
| SSL termination | Pass-through or terminate | Typically terminates |
| Performance | Faster — no payload inspection | Slower — parses request |
| Content-based routing | No | Yes — route `/api` vs `/static` to different pools |
| Session persistence | IP hash only | Cookie-based, header-based |
| Use case | Raw throughput, non-HTTP protocols | HTTP microservices, API gateways |
| Examples | AWS NLB, HAProxy (TCP mode) | AWS ALB, Nginx, Envoy |

### Top 6 Load Balancing Algorithms

| Algorithm | Type | How it works | Best for |
|-----------|------|-------------|----------|
| **Round Robin** | Static | Requests cycle through servers sequentially | Equal-capacity servers, stateless services |
| **Weighted Round Robin** | Static | Higher-weight servers get proportionally more requests | Mixed-capacity server fleet |
| **IP Hash** | Static | Hash of client IP determines server | Session affinity without cookies |
| **Least Connections** | Dynamic | Route to server with fewest active connections | Long-lived connections (WebSocket, DB pools) |
| **Least Response Time** | Dynamic | Route to server with fastest response + fewest connections | Heterogeneous backend performance |
| **Random** | Static | Randomly select a server | Simple, works surprisingly well at scale |

### Where to Place Load Balancers

```
Internet ──▶ [LB] ──▶ Web Servers ──▶ [LB] ──▶ App Servers ──▶ [LB] ──▶ DB Servers
               │                         │                         │
          (L7: HTTP)              (L7 or L4)                 (L4: TCP)
```

Place an LB between every tier that needs horizontal scale or failover.

## 4. Database Replication

### Leader–Follower Setup

```
        Writes
Client ────────▶ Leader (master)
                    │
            async replication
              ┌─────┼─────┐
              ▼     ▼     ▼
           Follower₁ F₂  F₃  ◀──── Reads
```

| Advantage | Detail |
|-----------|--------|
| **Performance** | Reads parallelized across followers; write load isolated to leader |
| **Reliability** | Data replicated — survives disk failure on any single node |
| **Availability** | Followers serve reads even if leader is briefly down |

### Failover Scenarios

| Scenario | Action | Risk |
|----------|--------|------|
| Follower goes down | Route reads to remaining followers; spin up replacement | Minimal — other followers absorb load |
| Leader goes down | Promote a follower to leader; redirect writes | **Replication lag** — promoted follower may be behind; data recovery scripts needed |
| All followers down | Leader temporarily serves reads + writes | Performance degradation; no redundancy |

Multi-leader and circular replication exist but add complexity (conflict resolution, split-brain). Default to single-leader unless you have multi-DC requirements.

## 5. CDN (Content Delivery Network)

### How It Works

```
User ──▶ CDN edge (closest PoP)
              │ cache HIT? ──▶ return asset
              │ cache MISS? ──▶ fetch from origin ──▶ cache + return
```

### When to Use CDN

| Use CDN | Don't bother |
|---------|-------------|
| Static assets (JS, CSS, images, video) | Infrequently accessed large files |
| Geographically distributed users | Single-region, low-traffic internal apps |
| High read-to-write ratio content | Highly dynamic per-user content |

### CDN Considerations

| Factor | Guidance |
|--------|----------|
| **Cost** | Charged per data transfer; exclude rarely accessed assets |
| **TTL** | Too short → frequent origin fetches; too long → stale content. Start 24h for static, 5min for semi-dynamic |
| **Invalidation** | API-based purge or **URL versioning** (`asset.js?v=2`) — prefer versioning |
| **Fallback** | App must handle CDN outage — serve from origin as fallback |
| **SSL** | CDN must support HTTPS; consider TLS termination at edge |

## 6. Stateless vs Stateful Architecture

| Dimension | Stateful | Stateless |
|-----------|----------|-----------|
| Session storage | In web server memory | External store (Redis, DB, cookie) |
| Sticky sessions | Required | Not needed |
| Auto-scaling | Hard — new nodes lack state | Easy — any node handles any request |
| Server failure | State lost with server | No state lost |
| Complexity | Simpler initial setup | Requires shared session store |

### How to Externalize Session State

```
                ┌─────────┐
Client ──▶ LB ──▶│ Web N   │──▶ Shared session store
                └─────────┘     (Redis / Memcached / DB)
                                     │
                              ┌──────┴──────┐
                              │ user_id: 42 │
                              │ role: admin  │
                              │ cart: [...]  │
                              └─────────────┘
```

**Stateless is the default choice for web tier.** Store session in:

| Store | Latency | Durability | Best for |
|-------|---------|-----------|----------|
| Redis / Memcached | ~1ms | Volatile (can persist) | Sessions, ephemeral state |
| RDBMS | ~5-10ms | Durable | Audit-critical session data |
| Signed cookies (JWT) | 0ms (client-side) | N/A | Small payloads, no server lookup |

## 7. Database Scaling

### Vertical vs Horizontal (Sharding)

| Dimension | Vertical | Horizontal (sharding) |
|-----------|----------|----------------------|
| Approach | Bigger machine | Split data across multiple machines |
| Limit | Hardware ceiling (e.g. 24TB RAM) | No hard limit |
| SPOF risk | High | Low per shard |
| Cost | Expensive at scale | Cost-effective |
| Complexity | Low | High — shard key selection, routing, resharding |
| Joins | Normal SQL joins | Cross-shard joins are expensive/impossible |
| Example | StackOverflow ran on 1 master DB until 10M+ monthly visitors | Most large social/commerce platforms |

### Sharding Strategies

| Strategy | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Hash-based** | `hash(shard_key) % N` | Even distribution | Resharding moves lots of data; use consistent hashing to mitigate |
| **Range-based** | Partition by value ranges (e.g. A-M, N-Z) | Simple; range queries within shard | Uneven distribution if ranges aren't balanced |
| **Directory-based** | Lookup table maps key → shard | Flexible; easy rebalancing | Lookup table is a SPOF and bottleneck |
| **Geo-based** | Partition by geographic region | Data locality; compliance (GDPR) | Cross-region queries are slow |

### Shard Key Selection Criteria

| Criterion | Why it matters |
|-----------|---------------|
| **High cardinality** | More possible values → more even distribution |
| **Low frequency skew** | No single value should dominate (avoid hotspots) |
| **Non-monotonic** | Auto-increment IDs cause all writes to hit latest shard |
| **Query alignment** | Key should match most common query patterns |

### Sharding Challenges

| Challenge | Description | Mitigation |
|-----------|------------|------------|
| **Resharding** | Adding/removing shards requires data movement | Consistent hashing; virtual buckets |
| **Celebrity/hotspot problem** | One shard gets disproportionate traffic (e.g. celebrity user) | Dedicated shard per hot key; further partition |
| **Cross-shard joins** | SQL joins across shards are very expensive | Denormalize; application-level joins |
| **Referential integrity** | Foreign keys can't span shards | Enforce in application layer |
| **Schema changes** | Must apply to all shards | Automated migration tooling |

### Request Routing

| Approach | How it works | Trade-off |
|----------|-------------|-----------|
| Shard-aware client | Client knows shard map | Fast; but client must stay in sync |
| Routing tier (proxy) | Central proxy routes to correct shard | Simple client; proxy can be bottleneck |
| Shard-aware node | Any node redirects to correct shard | No central proxy; gossip protocol overhead |

## 8. Message Queues for Decoupling

### Architecture

```
Producer(s) ──▶ [ Message Queue ] ──▶ Consumer(s)
   (web servers)   (Kafka, RabbitMQ,     (workers)
                    SQS, Redis Streams)
```

### When to Use

| Use message queues when | Don't use when |
|------------------------|----------------|
| Tasks are time-consuming (image processing, email) | Sub-millisecond latency required |
| Producer and consumer scale independently | Simple request-response is sufficient |
| You need to survive consumer downtime | Ordering guarantees are critical and complex |
| Workload is spiky — queue absorbs bursts | Payload is trivially small and fast |

### Benefits

| Benefit | Detail |
|---------|--------|
| **Decoupling** | Producer doesn't know or care about consumer implementation |
| **Async processing** | User gets fast response; heavy work happens in background |
| **Load leveling** | Queue absorbs traffic spikes; consumers process at their own rate |
| **Reliability** | Messages persist in queue even if consumers are temporarily down |
| **Independent scaling** | Scale consumers based on queue depth, not request rate |

### Basic Pattern: Photo Processing

```
Web Server ──POST /photo──▶ [Queue: photo-jobs]
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
               Worker 1        Worker 2        Worker N
              (crop/blur)    (crop/blur)     (crop/blur)
```

Queue large → add workers. Queue empty → reduce workers.

## 9. Multi-Data Center

### Architecture

```
                    ┌────────────┐
                    │  GeoDNS    │
                    └─────┬──────┘
               ┌──────────┼──────────┐
               ▼                     ▼
        ┌──────────┐          ┌──────────┐
        │ DC East  │◀────────▶│ DC West  │
        │ LB→Web→DB│  data    │ LB→Web→DB│
        │ Cache,MQ │  sync    │ Cache,MQ │
        └──────────┘          └──────────┘
```

### Key Challenges

| Challenge | Solution |
|-----------|----------|
| **Traffic routing** | GeoDNS routes users to nearest DC; failover to healthy DC on outage |
| **Data synchronization** | Async replication across DCs (e.g. Netflix multi-DC replication) |
| **Data consistency** | Eventual consistency model; conflict resolution for concurrent writes |
| **Cache consistency** | Independent cache per DC; invalidation propagation |
| **Deployment** | Automated deploy tools ensure identical versions across all DCs |
| **Testing** | Test at each DC location; synthetic traffic from multiple regions |

### Failover

| Scenario | Behavior |
|----------|----------|
| DC-East goes down | GeoDNS routes 100% traffic to DC-West |
| DC-East recovers | Gradually shift traffic back; verify data sync before full cutover |
| Network partition between DCs | Each DC serves local traffic; reconcile data when connectivity restores |

## Quick Reference: Scaling Checklist

When designing a system, walk through this checklist:

- [ ] **Web tier**: Stateless? Behind load balancer? Auto-scaling group?
- [ ] **Data tier**: Replication for reads? Sharding plan for growth?
- [ ] **Caching**: Cache-aside for hot data? TTL policy? Eviction strategy (LRU)?
- [ ] **Static assets**: On CDN? Versioned URLs for invalidation?
- [ ] **Async work**: Heavy tasks offloaded to message queue + workers?
- [ ] **Multi-region**: GeoDNS? Cross-DC data replication? Failover tested?
- [ ] **Redundancy**: No single points of failure at any tier?
- [ ] **Monitoring**: Logging, metrics, alerting at every tier?

See `references/scaling-fundamentals-detail.md` for deeper dives on sharding algorithms, LB configuration patterns, CDN strategies, replication topologies, and real-world case studies.
