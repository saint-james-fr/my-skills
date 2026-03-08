---
name: system-design-data-processing
description: Data processing system design — metrics monitoring and alerting, ad click aggregation, unique ID generation, real-time leaderboards, time-series data, and data pipelines. Use when designing monitoring systems, implementing data aggregation, generating distributed unique IDs, building leaderboards, or processing streaming data.
---

# Data Processing — System Design Reference

Based on *System Design Interview* (Vol. 1 & 2) by Alex Xu & Sahn Lam.

## 1. Unique ID Generation

### Approach Comparison

| Approach | Uniqueness | Sortable | Scalability | Availability | Pros | Cons |
|----------|-----------|----------|-------------|--------------|------|------|
| **Multi-master replication** | Unique (auto_increment by k) | Not across servers | Medium | Medium | Uses DB built-in feature | IDs not time-ordered across servers; hard to scale with add/remove servers |
| **UUID** | 128-bit, near-zero collision | No | High | High | No coordination needed; each server generates independently | 128-bit (not 64); not sortable by time; non-numeric |
| **Ticket server** | Centralized auto_increment | Yes (single server) | Low | Low (SPOF) | Numeric; simple to implement | Single point of failure; sync issues with multiple ticket servers |
| **Twitter Snowflake** | 64-bit, guaranteed unique | Yes (by timestamp) | High | High | Time-sortable; scalable; 64-bit | Clock sync required (NTP); limited to ~69 years per epoch |

### Snowflake Bit Layout (64 bits)

```
| Sign (1) | Timestamp (41) | Datacenter ID (5) | Machine ID (5) | Sequence (12) |
```

| Field | Bits | Range / Note |
|-------|------|-------------|
| Sign | 1 | Always 0 (reserved) |
| Timestamp | 41 | Milliseconds since custom epoch → ~69 years |
| Datacenter ID | 5 | 2^5 = 32 datacenters |
| Machine ID | 5 | 2^5 = 32 machines per datacenter |
| Sequence | 12 | 2^12 = 4096 IDs per millisecond per machine |

**Max throughput per machine:** 4096 IDs/ms = ~4M IDs/sec.

**Key considerations:**
- Clock synchronization via NTP is critical — clock drift can cause duplicate or out-of-order IDs
- Section length tuning: fewer sequence bits + more timestamp bits for low-concurrency / long-lived systems
- Datacenter and machine IDs are fixed at startup; changes require careful review to avoid conflicts

---

## 2. Metrics Monitoring & Alerting

### Data Model

Every time series consists of:

| Component | Type | Example |
|-----------|------|---------|
| Metric name | String | `cpu.load` |
| Tags / Labels | List of key:value pairs | `host:i631, env:prod` |
| Values + timestamps | Array of (value, timestamp) | `(0.29, 1613707265)` |

**Line protocol format:** `CPU.load host=webserver01,region=us-west 1613707265 50`

### Data Access Pattern

- **Write-heavy:** ~10M metrics written/day at high frequency
- **Reads are spiky:** dashboard refreshes and alert queries come in bursts
- Constant heavy writes + bursty reads

### Time-Series Database Selection

| Database | Type | Key Feature |
|----------|------|-------------|
| **InfluxDB** | Purpose-built TSDB | 250K+ writes/sec (8 cores, 32GB); Flux query language |
| **Prometheus** | Purpose-built TSDB | Pull-based; PromQL; strong ecosystem |
| **OpenTSDB** | TSDB on Hadoop/HBase | Distributed; requires Hadoop cluster |
| **TimescaleDB** | PostgreSQL extension | SQL-compatible time-series |

Why not a general-purpose DB? Moving average in SQL is complex nested queries. In Flux: `|> exponentialMovingAverage(size:-10s)`.

### Collection: Pull vs Push

| Aspect | Pull | Push |
|--------|------|------|
| **Example** | Prometheus | CloudWatch, Graphite |
| **Protocol** | TCP (HTTP `/metrics` endpoint) | Typically UDP (lower latency) |
| **Health check** | No response = server is down (easy detection) | Missing metrics could be network issue |
| **Short-lived jobs** | May miss short jobs (needs push gateway) | Naturally handles batch jobs |
| **Multi-datacenter** | Requires reachable endpoints (firewall issues) | LB + auto-scaling group accepts from anywhere |
| **Data authenticity** | Endpoints pre-defined in config — guaranteed authentic | Any client can push; needs whitelisting/auth |
| **Scaling** | Consistent hash ring maps each source to one collector | Auto-scaling collector cluster behind load balancer |

Large orgs typically support both (especially with serverless where no agent can be installed).

### Storage: Space Optimization

| Technique | How It Works |
|-----------|-------------|
| **Delta encoding** | Store base value + deltas: `1610087371, 10, 10, 9, 11` instead of full timestamps |
| **Compression** | Built-in to TSDBs (Gorilla compression for floats, run-length for timestamps) |
| **Downsampling** | 7 days: raw → 30 days: 1-min resolution → 1 year: 1-hour resolution |
| **Cold storage** | Move rarely-accessed old data to cheap storage (S3, Glacier) |

### Scaling the Pipeline

```
Metrics Sources → Metrics Collectors (auto-scaling) → Kafka → Stream Processors → Time-Series DB
```

Kafka benefits: decouples collection from processing; retains data if DB is down; partition by metric name for consumer parallelism.

### Alert System Flow

1. **Config rules** loaded into cache (YAML format: metric, threshold, duration, severity)
2. **Alert manager** fetches config, queries time-series DB at intervals
3. **Filter / merge / dedupe** alerts (e.g., merge within same instance)
4. **Alert store** (Cassandra) tracks state: inactive → pending → firing → resolved
5. Eligible alerts → **Kafka** → alert consumers → channels (email, PagerDuty, webhook)

### Aggregation Points

| Where | What | Trade-off |
|-------|------|-----------|
| **Collection agent** (client) | Simple counters pre-aggregated per minute | Reduces network volume; limited logic |
| **Ingestion pipeline** (stream) | Flink/Spark aggregation before writing | Significant write reduction; loses raw precision; late events are hard |
| **Query side** | Aggregate raw data at read time | No data loss; slower queries on large datasets |

---

## 3. Ad Click Aggregation

### System Scale

- 1B ad clicks/day → 10,000 QPS avg, 50,000 QPS peak
- 2M total ads; 30% YoY growth
- Daily storage: ~100 GB (0.1 KB per event)

### Data Model: Raw vs Aggregated

| | Raw Data | Aggregated Data |
|--|----------|----------------|
| **Pros** | Full data; supports filtering and recalculation | Smaller dataset; fast queries |
| **Cons** | Huge storage; slow queries | Derived data — loses detail |
| **Use** | Backup / debugging / recalculation; cold storage | Active queries / dashboards / billing |

**Store both.** Raw data serves as backup (cold storage for old data). Aggregated data serves active queries.

### High-Level Architecture

```
Log Files → Log Watcher → [Kafka #1] → Aggregation Service (MapReduce) → [Kafka #2] → DB Writer → Aggregation DB
                                                                                                  ↕
                                                                               Reconciliation (batch job)
```

Two Kafka queues: (1) raw click events, (2) aggregated results. Second queue enables end-to-end exactly-once semantics via atomic commit.

### MapReduce Aggregation (DAG Model)

| Node | Role | Example |
|------|------|---------|
| **Map** | Filter, transform, partition by ad_id | `ad_id % N` routes to aggregate nodes |
| **Aggregate** | Count clicks per ad_id in memory per minute | Maintains in-memory counters / heap |
| **Reduce** | Merge results from all aggregate nodes | Top N from each node → final top N |

**Use case 1 — click count per ad:** Map partitions by ad_id → each Aggregate counts → results stored.

**Use case 2 — top N ads:** each Aggregate maintains a heap (top N locally) → Reduce merges to global top N.

**Use case 3 — filtered aggregation:** pre-define filter dimensions (country, IP, user_id) → star schema → pre-computed buckets.

### Streaming vs Batching

| Dimension | Online Service | Batch System | Streaming System |
|-----------|---------------|--------------|-----------------|
| **Responsiveness** | Immediate response | No client response | No client response |
| **Input** | User requests | Bounded finite input | Unbounded infinite streams |
| **Output** | Responses | Materialized views, aggregated metrics | Materialized views, aggregated metrics |
| **Performance metric** | Availability, latency | Throughput | Throughput + latency |
| **Example** | Online shopping | MapReduce | Flink |

### Lambda vs Kappa Architecture

| Architecture | How It Works | Pros | Cons |
|-------------|-------------|------|------|
| **Lambda** | Batch layer + speed layer (streaming); merge results | Handles both real-time and historical reprocessing | Two codebases to maintain |
| **Kappa** | Single stream processing path; replay from log for reprocessing | Single codebase; simpler | Reprocessing throughput limited by stream engine |

### Aggregation Windows

| Window Type | Description | Use Case |
|-------------|------------|----------|
| **Tumbling** (fixed) | Non-overlapping fixed-size chunks | Aggregate clicks per minute |
| **Sliding** | Overlapping windows that slide by interval | Top N clicked ads in last M minutes |
| **Session** | Dynamic windows based on activity gaps | User session analysis |

### Data Recalculation

When a bug is found in aggregation:
1. Recalculation service reads from raw data storage (batch job)
2. Sends to **dedicated** aggregation service (does not impact real-time)
3. Aggregated results update DB via second message queue

---

## 4. Real-Time Gaming Leaderboard

### Approach Comparison

| Approach | Rank Query | Update | Scale | Limitation |
|----------|-----------|--------|-------|------------|
| **SQL** (`ORDER BY score DESC`) | O(n log n) table scan | Simple `UPDATE` / `INSERT` | Poor at millions of rows | 10+ seconds for rank query on large tables |
| **Redis sorted set** | O(log n) | O(log n) | Excellent (single node handles 25M users in ~650 MB) | Data in memory; needs persistence strategy |
| **NoSQL** (DynamoDB) | O(n/k) with write sharding | O(log n) per partition | Good with sharding | No straightforward exact rank; percentile-based |

### Redis Sorted Set Operations

| Operation | Command | Complexity | Description |
|-----------|---------|-----------|-------------|
| Add/update score | `ZADD key score member` | O(log n) | Insert or update member's score |
| Increment score | `ZINCRBY key increment member` | O(log n) | Atomically increment score; creates if absent |
| Get rank (desc) | `ZREVRANK key member` | O(log n) | 0-based rank from highest score |
| Top N players | `ZREVRANGE key 0 9 WITHSCORES` | O(log n + m) | Top 10 with scores |
| Relative position | `ZREVRANGE key start stop` | O(log n + m) | Players around a given rank |

**Workflow:**
1. Player wins → game service calls leaderboard service → `ZINCRBY leaderboard_feb_2021 1 'mary1934'`
2. Fetch top 10 → `ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES`
3. Fetch user rank → `ZREVRANK leaderboard_feb_2021 'mary1934'`
4. Fetch neighbors → get rank, then `ZREVRANGE leaderboard_feb_2021 (rank-4) (rank+4)`

### Scaling Redis (for 500M DAU)

| Strategy | How | Rank Calculation | Trade-off |
|----------|-----|-----------------|-----------|
| **Fixed partition** | Shard by score ranges (e.g., 1-100, 101-200, ...) | Local rank + count from higher shards (`info keyspace` is O(1)) | Need secondary cache for user→score mapping; handle cross-shard moves |
| **Hash partition** | `CRC16(key) % 16384` hash slots | Scatter-gather top N from each shard → merge | No straightforward exact rank; high latency with many partitions |

**Recommendation:** Fixed partition for exact ranking; hash partition if percentile ranking is acceptable.

### Tie-Breaking

Use Redis Hash to store `user_id → timestamp` of most recent win. On tie, the user who reached the score first ranks higher.

### Failure Recovery

MySQL records every win with timestamp. Recovery script iterates entries and replays `ZINCRBY` per user to rebuild the sorted set. Redis also supports persistence (RDB snapshots, AOF) and read replicas for fast failover.

---

## 5. Data Pipelines

### Pipeline Stages

```
Collect → Ingest → Process → Store → Serve
```

| Stage | Role | Technologies |
|-------|------|-------------|
| **Collect** | Gather data from sources (logs, events, APIs) | Fluentd, Logstash, agents |
| **Ingest** | Buffer and transport reliably | Kafka, Kinesis, Pulsar |
| **Process** | Transform, aggregate, enrich | Flink, Spark, Kafka Streams |
| **Store** | Persist for queries and analytics | TSDB, Cassandra, S3, data warehouse |
| **Serve** | Query, visualize, alert | Grafana, dashboards, APIs |

### Batch vs Stream Processing

| Dimension | Batch | Stream |
|-----------|-------|--------|
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Data** | Bounded, finite | Unbounded, infinite |
| **Throughput** | Very high | High (but lower per-event) |
| **Fault tolerance** | Retry entire batch | Checkpoint + replay |
| **Use case** | ETL, daily reports, reconciliation | Real-time dashboards, alerts, fraud detection |
| **Technology** | Spark, MapReduce, Hive | Flink, Kafka Streams, Spark Streaming |

---

## 6. Delivery Guarantees in Data Processing

| Guarantee | Behavior | Use Case | Risk |
|-----------|---------|----------|------|
| **At-most once** | Fire and forget; no retry | Metrics collection where occasional loss is OK | Data loss |
| **At-least once** | Retry on failure; may duplicate | Most stream processing | Duplicates need handling |
| **Exactly once** | Distributed transaction (atomic commit) | Billing, ad click aggregation | Complex; higher latency |

### Deduplication Strategies

| Source | Technique |
|--------|-----------|
| **Client-side duplicates** | Idempotency keys; ad fraud/risk control |
| **Server outage duplicates** | Track Kafka offsets externally; only process if offset matches last saved |
| **Exactly-once processing** | Atomic commit: save offset + send downstream in single distributed transaction |

### Watermarks for Late-Arriving Data

- **Watermark** = extended aggregation window (e.g., +15 seconds) to catch slightly delayed events
- Long watermark → better accuracy, higher latency
- Short watermark → lower latency, less accurate
- Events arriving after watermark → corrected by end-of-day reconciliation batch job

---

## 7. Hotspot Handling

### Detection

- Monitor per-partition / per-node metrics: CPU, queue size, records-lag
- Hot keys = ads or entities with disproportionately high traffic (major advertisers)

### Mitigation

| Technique | How It Works |
|-----------|-------------|
| **Split hot aggregation nodes** | Hot node requests extra resources via resource manager → splits events across multiple child nodes → merges results back |
| **Dedicated processing** | Route known hot keys to dedicated, higher-capacity nodes |
| **Global-Local aggregation** | Pre-aggregate locally on each node, then merge globally (reduces per-node hot-key impact) |
| **Split Distinct aggregation** | Distribute distinct counting across multiple nodes before combining |

### Kafka Partitioning for Hot Keys

- Pre-allocate enough partitions to avoid runtime rebalancing
- Use `ad_id` as hashing key so all events for an ad land in the same partition
- For very hot keys, further split within the aggregation layer (not at Kafka level)

---

## 8. Monitoring & Reconciliation

### End-to-End Monitoring

| Metric | Why |
|--------|-----|
| **Latency per stage** | Track timestamps as events flow through pipeline; expose as latency metrics |
| **Queue size / records-lag** | Sudden increase → add more consumers or aggregation nodes |
| **System resources** | CPU, disk, JVM heap on aggregation nodes |
| **Data completeness** | Compare expected event count vs received count |

### Reconciliation

- **Batch reconciliation:** end-of-day (or hourly) batch job sorts raw events by event time, re-aggregates, and compares against streaming results
- Results may not match exactly due to late-arriving events (acceptable within watermark tolerance)
- Discrepancies trigger alerts and potential recalculation from raw data

### Alert Fatigue Management

| Technique | Description |
|-----------|-------------|
| **Alert merging** | Combine alerts from the same instance/metric within a time window |
| **Severity levels** | page (critical) vs warning vs info — only page for actionable issues |
| **Deduplication** | Suppress repeated alerts for the same ongoing incident |
| **Access control** | Route alerts to relevant on-call teams only |
| **Threshold tuning** | Adjust thresholds and durations (`for: 5m`) to reduce noise |

---

## Quick Reference: When to Use What

| Problem | Solution |
|---------|----------|
| Need time-sortable 64-bit IDs | Snowflake ID generator |
| Infrastructure health monitoring | TSDB + pull/push collectors + Grafana + alerting |
| Aggregate billions of events/day | MapReduce DAG (Map → Aggregate → Reduce) + Kafka |
| Real-time ranking with millions of users | Redis sorted sets (ZADD, ZINCRBY, ZREVRANK) |
| Process unbounded event streams | Kappa architecture with Flink/Kafka Streams |
| Ensure billing-grade accuracy | Exactly-once delivery + watermarks + batch reconciliation |
| Handle hot partitions | Split aggregation nodes + Global-Local aggregation |
