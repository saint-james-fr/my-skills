---
name: system-design-interview-framework
description: The 4-step system design interview framework, back-of-the-envelope estimation techniques, and trade-off analysis. Use when approaching a system design problem, estimating capacity/QPS/storage, structuring a design discussion, or evaluating architectural trade-offs.
---

# System Design Interview Framework

Based on *System Design Interview* (Alex Xu) and ByteByteGo references.

## 1. The 4-Step Framework

| Step | Activity | Time (45 min) |
|------|----------|---------------|
| 1 | Understand the problem & establish design scope | 3-10 min |
| 2 | Propose high-level design & get buy-in | 10-15 min |
| 3 | Design deep dive (pick 2-3 components) | 10-25 min |
| 4 | Wrap up (bottlenecks, improvements, summary) | 3-5 min |

### Step 1 — Understand the Problem & Establish Design Scope

Goal: clarify functional requirements, non-functional requirements, and constraints before drawing anything.

**Clarifying questions to ask:**

| Category | Questions |
|----------|-----------|
| Users & scale | DAU/MAU? Read-heavy or write-heavy? Expected QPS? |
| Features | What specific features are we building? What is out of scope? |
| Platforms | Mobile, web, or both? API-only? |
| Non-functional | Latency requirements? Availability target? Consistency model? |
| Constraints | Existing tech stack? Can we use managed services? |
| Data | Data size per record? Retention period? Growth rate? |
| Edge cases | What happens at peak traffic? International users? |

**Output of Step 1:** written list of functional requirements, non-functional requirements, and back-of-the-envelope estimates.

### Step 2 — Propose High-Level Design & Get Buy-In

Goal: draw a blueprint with key components and walk through 1-2 core use cases.

**Components to consider:**

| Layer | Typical components |
|-------|-------------------|
| Client | Web app, mobile app, third-party API consumers |
| Edge | DNS, CDN, load balancer, API gateway |
| Application | Web servers, app servers, microservices |
| Data | SQL/NoSQL database, cache (Redis/Memcached), object storage (S3) |
| Async | Message queue (Kafka, SQS), task workers |
| Infra | Logging, monitoring, alerting |

**What to include:**

- API design (key endpoints, request/response shape)
- Data model (core tables/collections, relationships, sharding key candidates)
- High-level architecture diagram (boxes + arrows)
- Walk through 1-2 concrete use cases end-to-end

**When to include API endpoints / DB schema:** always for focused problems (URL shortener, chat system). Skip for very broad problems (design Google search) unless interviewer asks.

### Step 3 — Design Deep Dive

Goal: pick 2-3 components the interviewer cares about and go deep.

**How to choose what to dive into:**

| Signal | Action |
|--------|--------|
| Interviewer hints at an area | Follow their lead |
| Obvious bottleneck in your design | Address it proactively |
| Novel/interesting algorithmic challenge | Propose and discuss alternatives |
| Senior-level interview | Focus on scalability, fault tolerance, performance |

**Common deep-dive topics:**

- Database schema + indexing strategy + sharding
- Cache strategy (read-through, write-through, write-behind, cache-aside)
- Consistency model (strong vs eventual, conflict resolution)
- Rate limiting / abuse prevention
- Data partitioning and replication
- Search / ranking algorithms
- Notification / real-time delivery (WebSocket, SSE, long polling)

### Step 4 — Wrap Up

Goal: leave a strong final impression.

**Wrap-up checklist:**

- Identify remaining bottlenecks and propose mitigations
- Recap the design end-to-end (30-second summary)
- Discuss error handling (server failure, network partition, data corruption)
- Mention operational concerns (monitoring, alerting, deployment, rollback)
- Propose next-scale evolution (current design handles X; for 10X, we would...)
- Suggest refinements if given more time

### Dos and Don'ts

| Do | Don't |
|----|-------|
| Ask clarifying questions before designing | Jump into a solution immediately |
| State assumptions explicitly | Assume the interviewer agrees with unstated assumptions |
| Think out loud, narrate your reasoning | Think in silence |
| Drive the discussion, suggest multiple approaches | Wait passively for direction |
| Design most critical components first | Spend too long on a single component early on |
| Treat the interviewer as a teammate | Ignore feedback or hints |
| Acknowledge trade-offs in every decision | Claim your design is perfect |
| Manage time across all 4 steps | Spend 30 min on Step 1 |

---

## 2. Back-of-the-Envelope Estimation

### Power of 2 — Data Volume Quick Reference

| Power | Approx. Value | Name | Short |
|-------|---------------|------|-------|
| 10 | 1 Thousand | Kilobyte | 1 KB |
| 20 | 1 Million | Megabyte | 1 MB |
| 30 | 1 Billion | Gigabyte | 1 GB |
| 40 | 1 Trillion | Terabyte | 1 TB |
| 50 | 1 Quadrillion | Petabyte | 1 PB |

### Latency Numbers Every Programmer Should Know

| Operation | Latency | Order of magnitude |
|-----------|---------|-------------------|
| L1 cache reference | 0.5 ns | ns |
| Branch mispredict | 5 ns | ns |
| L2 cache reference | 7 ns | ns |
| Mutex lock/unlock | 100 ns | ns |
| Main memory reference | 100 ns | ns |
| Compress 1 KB (Zippy/Snappy) | 10 us | us |
| Send 2 KB over 1 Gbps network | 20 us | us |
| Read 1 MB sequentially from memory | 250 us | us |
| Round trip within same datacenter | 500 us | us |
| Disk seek | 10 ms | ms |
| Read 1 MB sequentially from network | 10 ms | ms |
| Read 1 MB sequentially from disk | 30 ms | ms |
| Send packet CA -> Netherlands -> CA | 150 ms | ms |

**Key takeaways:** memory is fast, disk is slow. Avoid disk seeks. Compress before sending over network. Cross-region round trips are expensive.

### Availability Numbers (SLA Nines)

| Availability | Downtime/day | Downtime/year |
|--------------|-------------|---------------|
| 99% (two nines) | 14.40 min | 3.65 days |
| 99.9% (three nines) | 1.44 min | 8.77 hours |
| 99.99% (four nines) | 8.64 sec | 52.60 min |
| 99.999% (five nines) | 864 ms | 5.26 min |
| 99.9999% (six nines) | 86.4 ms | 31.56 sec |

### QPS Estimation Technique

Formula:

```
DAU = MAU * daily_active_ratio
QPS = DAU * actions_per_user / 86400
Peak QPS = QPS * peak_multiplier  (typically 2x-5x)
```

**Worked example — Twitter-like service:**

```
MAU              = 300M
Daily active %   = 50%  ->  DAU = 150M
Tweets/user/day  = 2
QPS              = 150M * 2 / 86400 ≈ 3,500
Peak QPS         = 2 * 3,500 = ~7,000
```

### Storage Estimation Technique

Formula:

```
Daily new data     = DAU * actions_per_user * avg_size_per_action
Yearly storage     = daily_new_data * 365
N-year storage     = yearly * N
```

**Worked example — Twitter media storage:**

```
Tweets with media  = 150M * 2 * 10% = 30M media items/day
Avg media size     = 1 MB
Daily storage      = 30M * 1 MB = 30 TB/day
5-year storage     = 30 TB * 365 * 5 ≈ 55 PB
```

### Estimation Tips

| Tip | Why |
|-----|-----|
| Round aggressively (100,000 / 10 not 99,987 / 9.1) | Precision doesn't matter; process does |
| Write down assumptions | You'll reference them later |
| Label units (KB, MB, QPS) | Prevents confusion and shows rigor |
| Sanity-check results | "30 TB/day sounds right for a media platform" |

### Handy Conversion Shortcuts

| Fact | Value |
|------|-------|
| Seconds in a day | ~86,400 ≈ ~10^5 |
| Seconds in a month | ~2.5M ≈ ~2.5 * 10^6 |
| Seconds in a year | ~31.5M ≈ ~3 * 10^7 |
| 1 million requests/day | ~12 QPS |
| 1 billion requests/day | ~12,000 QPS |
| 1 char (ASCII) | 1 byte |
| 1 UUID | 128 bits = 16 bytes |
| 1 timestamp (epoch) | 4-8 bytes |
| Typical metadata per record | ~500 bytes - 1 KB |
| Average image | 200 KB - 500 KB |
| Average video (1 min) | 50 MB - 150 MB |

---

## 3. Trade-Off Analysis

Everything is a trade-off. There is no right or wrong design — only designs that fit the requirements better.

### 10 Core Trade-Offs

| Trade-off | Option A | Option B | When to prefer A | When to prefer B |
|-----------|----------|----------|-------------------|-------------------|
| Vertical vs Horizontal Scaling | Scale up (bigger machine) | Scale out (more machines) | Low traffic, simplicity | High traffic, fault tolerance |
| SQL vs NoSQL | Relational (ACID, joins) | Non-relational (flexible schema) | Complex queries, transactions | Massive scale, unstructured data |
| Consistency vs Availability | CP (reject requests if inconsistent) | AP (serve stale data, stay available) | Financial systems, inventory | Social feeds, analytics |
| Strong vs Eventual Consistency | Immediate read-after-write | Data propagates asynchronously | User-facing writes (profile update) | Counters, feeds, logs |
| Normalization vs Denormalization | Normalized (no redundancy) | Denormalized (redundant, fast reads) | Write-heavy, data integrity | Read-heavy, query performance |
| Batch vs Stream Processing | Process data periodically | Process data in real time | Reporting, ETL, cost-sensitive | Fraud detection, live dashboards |
| Stateful vs Stateless | Server remembers client state | Server keeps no state | WebSocket connections | REST APIs, horizontal scaling |
| Push vs Pull | Server pushes updates to client | Client polls for updates | Real-time needs (chat, notifications) | Infrequent updates, simple infra |
| Read-Through vs Write-Through Cache | Load into cache on miss | Write to cache + DB simultaneously | Read-heavy workloads | Write-heavy, consistency matters |
| Sync vs Async Processing | Blocking, sequential | Non-blocking, background | Simple flows, low latency needed | Long tasks, decoupled services |

### Additional Trade-Offs to Mention

| Dimension | Tension |
|-----------|---------|
| Cost vs Performance | Spend more for lower latency, or optimize cost and accept higher p99 |
| Reliability vs Scalability | More replicas = more reliable but harder to keep consistent at scale |
| Security vs Flexibility | Stricter auth/authz = more secure but slower development iteration |
| Development Speed vs Quality | Ship fast with shortcuts, or invest in tests/abstractions upfront |
| Latency vs Throughput | Optimize for single-request speed or total requests processed |
| Monolith vs Microservices | Simple deployment vs independent scaling + team autonomy |
| REST vs GraphQL | Simplicity + caching vs flexible queries + reduced over-fetching |

### How to Present Trade-Offs in an Interview

1. State both options clearly
2. Name the axis of tension (e.g., "this is a consistency vs latency trade-off")
3. Explain which option fits the current requirements and why
4. Acknowledge what you're giving up

---

## 4. Non-Functional Requirements Checklist

Use this checklist during Step 1 to probe which non-functional requirements matter most.

| Requirement | Key question | Typical target |
|-------------|-------------|----------------|
| Scalability | How many users / requests must we handle? | 10K-1B+ DAU |
| Availability | What uptime SLA? | 99.9% - 99.999% |
| Consistency | Can we tolerate stale reads? | Strong / eventual / causal |
| Latency | What is acceptable p50/p99? | p99 < 200ms (read), < 500ms (write) |
| Durability | Can we lose data? | Zero data loss for payments; best-effort for analytics |
| Security | Auth, encryption, compliance? | OAuth2, TLS, GDPR/SOC2 |
| Cost | Budget constraints? Managed vs self-hosted? | Depends on stage and scale |
| Maintainability | Team size? On-call burden? | Small team favors managed services |
| Observability | Logging, metrics, tracing? | Structured logs, p50/p99 dashboards, distributed tracing |

---

## 5. Scaling Building Blocks Quick Reference

| Technique | What it does | When to use |
|-----------|-------------|-------------|
| Load balancer | Distributes traffic across servers | Multiple app servers |
| CDN | Caches static assets at edge | Global users, static content |
| Cache (Redis/Memcached) | Stores hot data in memory | Read-heavy, reduce DB load |
| Database replication | Master-slave read replicas | Read-heavy, high availability |
| Database sharding | Horizontal data partitioning | Data too large for one node |
| Message queue | Async processing, decoupling | Background jobs, event-driven |
| Consistent hashing | Even key distribution across nodes | Distributed cache, sharding |
| Rate limiter | Throttle abusive traffic | API protection, DDoS mitigation |
| Blob/object storage | Store large binary objects (S3) | Images, videos, backups |
| Search index | Full-text and faceted search (ES) | Search features, autocomplete |

---

## Detailed Reference

For worked estimation examples, extended dos/don'ts, clarifying question templates by system type, and a full answer structuring template, see [interview-framework-detail.md](references/interview-framework-detail.md).
