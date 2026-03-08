# System Design Interview Framework — Detailed Reference

Extended examples, templates, and guidance for the framework in [SKILL.md](../SKILL.md).

---

## 1. Worked Estimation Examples

### Example A — Twitter-like Service (QPS + Storage)

**Assumptions:**

| Parameter | Value |
|-----------|-------|
| MAU | 300M |
| Daily active ratio | 50% |
| DAU | 150M |
| Tweets per user per day | 2 |
| Tweets containing media | 10% |
| Average text tweet size | ~200 bytes (id + text + metadata) |
| Average media size | 1 MB |
| Retention period | 5 years |

**QPS:**

```
Tweet QPS     = 150M * 2 / 86,400 ≈ 3,500
Peak QPS      = 2 * 3,500 ≈ 7,000
Read QPS      = assume 10:1 read:write → ~35,000 read QPS
Peak read QPS = ~70,000
```

**Storage:**

```
Text storage/day   = 150M * 2 * 200 bytes = 60 GB/day
Media storage/day  = 150M * 2 * 10% * 1 MB = 30 TB/day
5-year text        = 60 GB * 365 * 5 ≈ 110 TB
5-year media       = 30 TB * 365 * 5 ≈ 55 PB
```

**Bandwidth:**

```
Outgoing (reads)   = 70,000 QPS * 200 bytes (text) ≈ 14 MB/s text
                   + media reads → much higher, served from CDN
```

---

### Example B — YouTube-like Service (Storage + Bandwidth)

**Assumptions:**

| Parameter | Value |
|-----------|-------|
| DAU | 500M |
| Videos watched per user per day | 5 |
| Average video duration | 5 min |
| Average video file size (compressed) | 150 MB |
| Videos uploaded per day | 500K |
| Video retention | indefinite |

**Storage:**

```
Daily upload storage = 500K * 150 MB = 75 TB/day
Yearly storage       = 75 TB * 365 ≈ 27 PB/year
(Multiple resolutions: 3-4x raw → ~100-300 PB/year with transcoding variants)
```

**Bandwidth:**

```
Video views/day   = 500M * 5 = 2.5B views/day
Streaming QPS     = 2.5B / 86,400 ≈ 29,000 concurrent streams
Outbound bandwidth = 29,000 * ~5 Mbps avg bitrate ≈ 145 Gbps
```

---

### Example C — Chat System (QPS + Connections)

**Assumptions:**

| Parameter | Value |
|-----------|-------|
| DAU | 50M |
| Messages sent per user per day | 40 |
| Average message size | 200 bytes |
| Peak concurrent connections | 10% of DAU |
| Message retention | 1 year |

**QPS:**

```
Message QPS      = 50M * 40 / 86,400 ≈ 23,000
Peak QPS         = 3 * 23,000 ≈ 70,000
```

**Concurrent connections:**

```
Peak connections = 50M * 10% = 5M WebSocket connections
Per server (10K connections/server) → 500 servers for WebSocket tier
```

**Storage:**

```
Daily storage    = 50M * 40 * 200 bytes = 400 GB/day
Yearly storage   = 400 GB * 365 ≈ 146 TB/year
```

---

## 2. Extended Dos and Don'ts

### Dos — With Reasoning

| Do | Reasoning |
|----|-----------|
| Ask clarifying questions first | Shows you don't make assumptions; mirrors real engineering work |
| Write down assumptions and estimates | Creates shared context; you can reference them when making decisions |
| Think out loud continuously | Interviewer can only evaluate what they hear; silence = no signal |
| Start with high-level design, then zoom in | Prevents tunnel vision on one component |
| Propose multiple approaches, then pick one | Demonstrates breadth and decision-making ability |
| Acknowledge trade-offs explicitly | Shows maturity; real systems are always about trade-offs |
| Check in with the interviewer frequently | Ensures alignment; avoids 15 min wasted on something they don't care about |
| Manage your own time | Nobody will stop you; running out of time before deep dive is a red flag |
| Use concrete numbers | "~10K QPS" is better than "a lot of traffic" |
| Circle back to requirements when making decisions | Grounds every choice in the problem context |

### Don'ts — With Reasoning

| Don't | Why |
|-------|-----|
| Jump into a solution immediately | Biggest red flag; suggests you don't scope problems |
| Over-engineer from the start | Designing for 1B users when the problem says 10K is a negative signal |
| Go deep on one component too early | Lose time budget; other steps get rushed |
| Use buzzwords without explaining them | "We use Kafka" means nothing without explaining why |
| Ignore the interviewer's hints | They're steering you toward what they want to evaluate |
| Claim the design is perfect | No design is perfect; shows lack of self-awareness |
| Design in silence | Interviewer has no signal; awkward for everyone |
| Skip error handling and failure modes | Production systems fail; ignoring this shows inexperience |
| Forget to discuss data model | Data outlives code; schema design is critical |
| Mention a technology you can't explain | Interviewer will probe; bluffing backfires |

---

## 3. Common Clarifying Questions by System Type

### URL Shortener

| Question | Why it matters |
|----------|---------------|
| What is the expected URL volume? | Determines storage, QPS, and hash space |
| Custom short URLs allowed? | Changes API design and uniqueness constraints |
| How long do URLs live? Expiration? | Affects storage growth and cleanup strategy |
| Analytics required (click count, referrer)? | Adds read path complexity |
| What is the read:write ratio? | Drives caching strategy |

### Chat System

| Question | Why it matters |
|----------|---------------|
| 1:1 only or group chats too? Max group size? | Fan-out strategy changes dramatically |
| Online/offline status required? | Adds presence service and heartbeat infra |
| Message persistence and history? | Storage tier design |
| Media support (images, files)? | Adds upload service and object storage |
| End-to-end encryption? | Changes key management and message routing |
| Read receipts and typing indicators? | Additional real-time events |

### News Feed

| Question | Why it matters |
|----------|---------------|
| Reverse chronological or ranked? | Ranking adds ML scoring pipeline |
| How many friends/followers per user? | Fan-out cost at write time vs read time |
| Media types supported? | Impacts storage and CDN needs |
| How fresh must the feed be? | Eventual consistency tolerance |
| Ads in feed? | Separate ad-serving system integration |

### Notification System

| Question | Why it matters |
|----------|---------------|
| Push, SMS, email, or all? | Each channel has different infra |
| Real-time or batched? | Throughput vs latency design |
| Can users set preferences? | Adds preference service and filtering logic |
| Rate limiting per user? | Prevents notification spam |
| Retry policy on failure? | Durability guarantees |

### Rate Limiter

| Question | Why it matters |
|----------|---------------|
| Client-side or server-side? | Scope of the design |
| Throttle by IP, user ID, or API key? | Key design for the counter |
| What are the throttle rules? | Determines algorithm choice |
| Distributed or single-server? | Adds shared state (Redis) |
| Should throttled users be informed? | Response format (429 + headers) |

### Search / Autocomplete

| Question | Why it matters |
|----------|---------------|
| What is the corpus size? | Index size and sharding strategy |
| Latency requirement for suggestions? | p99 < 100ms drives aggressive caching |
| Personalized results? | Adds user-context scoring |
| Multi-language support? | Tokenization and stemming complexity |
| How often does the corpus change? | Real-time indexing vs batch rebuild |

---

## 4. Template for Structuring a Design Answer

Use this as a mental scaffolding during the interview:

### Phase 1: Scope (3-10 min)

```
1. Restate the problem in your own words
2. List functional requirements (bullet points)
3. List non-functional requirements (bullet points)
4. State assumptions and constraints
5. Do back-of-the-envelope estimation if relevant
   - DAU, QPS, peak QPS
   - Storage per day, per year
   - Bandwidth
```

### Phase 2: High-Level Design (10-15 min)

```
1. Define core API endpoints
   - POST /api/v1/resource
   - GET  /api/v1/resource/:id
   
2. Define data model
   - Core entities, fields, types
   - Primary keys, indexes
   - Relationships (1:1, 1:N, N:N)
   
3. Draw architecture diagram
   - Client → LB → App Servers → DB
   - Add: cache, CDN, message queue, workers as needed
   
4. Walk through 1-2 use cases end-to-end
   - "When a user creates X, here's what happens..."
   - "When a user reads Y, the request flows through..."
```

### Phase 3: Deep Dive (10-25 min)

```
1. Pick 2-3 components to go deep on
   (follow interviewer signals)

2. For each component, discuss:
   - Algorithm or approach chosen
   - Why this approach over alternatives (trade-offs)
   - How it scales (partitioning, replication)
   - Failure modes and mitigation
   - Performance characteristics (latency, throughput)

3. Common deep-dive areas:
   - Database: schema, indexes, sharding key, replication topology
   - Cache: strategy, invalidation, thundering herd protection
   - Consistency: conflict resolution, consensus protocol
   - Real-time: WebSocket fan-out, presence, ordering guarantees
```

### Phase 4: Wrap Up (3-5 min)

```
1. Summarize the design in 30 seconds
2. Call out remaining bottlenecks
3. Propose solutions for those bottlenecks
4. Discuss operational concerns
   - Monitoring: key metrics to track
   - Alerting: what triggers a page
   - Deployment: rolling update, blue-green, canary
5. Mention what you'd add with more time
   - Regional failover
   - Detailed security review
   - Cost optimization
```

---

## 5. System Design Checklist (15 Dimensions)

Use this as a final review before wrapping up your answer:

| # | Dimension | Key consideration |
|---|-----------|-------------------|
| 1 | Requirement gathering | Did I confirm functional + non-functional reqs? |
| 2 | System architecture | Are components well-defined with clear responsibilities? |
| 3 | Data design | Schema, storage engine, sharding, replication? |
| 4 | API design | Endpoints, methods, request/response format? |
| 5 | Scalability | Can each tier scale independently? |
| 6 | Reliability | What happens when a component fails? |
| 7 | Availability | SLA target? Redundancy at every tier? |
| 8 | Performance | Caching? CDN? DB indexing? |
| 9 | Consistency | What consistency model? Conflict resolution? |
| 10 | Security | Authentication, authorization, encryption? |
| 11 | Maintainability | Can the team operate this at 3 AM? |
| 12 | Cost | Is this cost-effective for the current scale? |
| 13 | Observability | Logging, metrics, tracing, alerting? |
| 14 | Testing | How would you test this at scale? |
| 15 | Migration / Rollout | How to deploy without downtime? |

---

## 6. Quick Estimation Cheat Sheet

Keep these in your head for rapid estimation:

| Metric | Quick formula |
|--------|--------------|
| QPS from DAU | DAU * actions / 86,400 |
| Peak QPS | 2x - 5x average QPS |
| Storage/day | DAU * actions * avg_size |
| Bandwidth | QPS * avg_response_size |
| Servers needed | Peak QPS / QPS_per_server (typically 1K-10K QPS/server) |
| Cache size | Working set * avg_object_size (typically 20% of data is hot) |
| Read replicas | Read QPS / reads_per_replica |

### Common Scale Anchors

| Scale | DAU | QPS (approx) | Example |
|-------|-----|-------------|---------|
| Small | 100K | ~10 | Internal tool |
| Medium | 10M | ~1,000 | Regional app |
| Large | 100M | ~10,000 | National platform |
| Massive | 1B+ | ~100,000+ | Global social network |
