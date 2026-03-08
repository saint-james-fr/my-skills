# Data Processing — Detailed Reference

Extended reference for topics covered in the SKILL.md.

---

## 1. Snowflake ID: Implementation Details & Clock Synchronization

### Bit Layout Deep Dive

```
 0                   1                   2                   3                   4                   5                   6
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                         Timestamp (41 bits)                         | DC (5) | Machine (5)|   Sequence (12)   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### ID Generation Algorithm

```
function generateId():
    timestamp = currentTimeMillis() - CUSTOM_EPOCH

    if timestamp < lastTimestamp:
        throw ClockMovedBackwardsError

    if timestamp == lastTimestamp:
        sequence = (sequence + 1) & 0xFFF   // 4095 mask
        if sequence == 0:
            timestamp = waitNextMillis(lastTimestamp)
    else:
        sequence = 0

    lastTimestamp = timestamp

    return (timestamp << 22)
         | (datacenterId << 17)
         | (machineId << 12)
         | sequence
```

### Clock Synchronization Challenges

| Problem | Impact | Mitigation |
|---------|--------|-----------|
| **Clock drift between servers** | Duplicate IDs if two servers share same datacenter+machine bits | NTP synchronization; monitor clock offset |
| **Clock moves backwards** (NTP correction, leap second) | Generator must refuse or wait | Detect and either throw error or wait until clock catches up |
| **Multi-core timestamp** | Same timestamp on different cores | Per-core sequence or lock on sequence increment |
| **NTP jitter** | Sub-millisecond ordering not guaranteed | Accept that ordering within the same millisecond is arbitrary |

### Section Length Tuning

| Configuration | Timestamp | DC | Machine | Sequence | Max Lifespan | Max IDs/ms/machine |
|--------------|-----------|-----|---------|----------|-------------|-------------------|
| **Default** | 41 | 5 | 5 | 12 | ~69 years | 4,096 |
| **High concurrency** | 39 | 5 | 5 | 14 | ~17 years | 16,384 |
| **Long lifespan** | 43 | 4 | 4 | 12 | ~278 years | 4,096 |
| **Few datacenters** | 41 | 2 | 8 | 12 | ~69 years | 4,096 (256 machines) |

### Epoch Selection

- Twitter Snowflake default epoch: `1288834974657` (Nov 04, 2010, 01:42:54 UTC)
- Choose a custom epoch close to your system's launch date to maximize the 69-year window
- Epoch change after deployment requires an ID migration strategy

### Comparison with Other Distributed ID Schemes

| Scheme | Size | Sortable | Coordination | Notes |
|--------|------|----------|-------------|-------|
| **Snowflake** | 64-bit | Yes (time) | Machine ID assignment | Industry standard |
| **UUID v4** | 128-bit | No | None | Random; no coordination needed |
| **UUID v7** | 128-bit | Yes (time) | None | Time-ordered UUID; newer standard |
| **ULID** | 128-bit | Yes (time) | None | Lexicographically sortable; Crockford base32 |
| **KSUID** | 160-bit | Yes (time) | None | 32-bit timestamp + 128-bit random |
| **MongoDB ObjectId** | 96-bit | Yes (time) | Machine + process ID embedded | 4-byte timestamp + 5-byte random + 3-byte counter |

---

## 2. Time-Series DB Internals

### Why TSDBs Outperform General-Purpose Databases

| Feature | TSDB | RDBMS |
|---------|------|-------|
| Write optimization | Append-only / LSM tree; batched writes | B-tree; random I/O on index updates |
| Compression | Domain-specific: delta-of-delta for timestamps, XOR for floats (Gorilla) | Generic page compression |
| Query language | Purpose-built (PromQL, Flux, InfluxQL) | SQL (complex for time-series operations) |
| Retention policies | Built-in TTL and downsampling | Manual implementation required |
| Label/tag indexing | Native inverted index on labels | Requires additional indexes |

### LSM Tree Architecture (used by InfluxDB, Prometheus)

```
Write Path:
  Event → WAL (Write-Ahead Log) → In-Memory Table (MemTable)
                                        ↓ (when full)
                                   Flush to SSTable (Sorted String Table) on disk
                                        ↓ (background)
                                   Compaction: merge SSTables, remove tombstones

Read Path:
  Query → MemTable → Level 0 SSTables → Level 1 → ... → Level N
         (Bloom filters skip irrelevant SSTables)
```

### Gorilla Compression (Facebook)

Used by Gorilla in-memory TSDB and adopted by InfluxDB, Prometheus.

**Timestamp compression (delta-of-delta):**

| Step | Value | Stored As |
|------|-------|-----------|
| Base | `1610087371` | Full 32 bits |
| T+1 | `1610087381` | Delta: `10` (4 bits) |
| T+2 | `1610087391` | Delta-of-delta: `0` (1 bit) |
| T+3 | `1610087400` | Delta-of-delta: `-1` (few bits) |

For regular intervals (e.g., every 10s), delta-of-delta is often 0, compressible to 1 bit per timestamp.

**Value compression (XOR encoding):**
- XOR consecutive float values
- If values are similar, most bits are zero → encode only the differing bits
- Typical compression: 1.37 bytes per data point (vs 16 bytes uncompressed)

### Retention and Downsampling Pipeline

```
Incoming data (raw, 10s intervals)
  ↓ 7 days
Downsample to 1-minute resolution
  ↓ 30 days
Downsample to 1-hour resolution
  ↓ 1 year
Move to cold storage (S3/Glacier)
  ↓ configurable
Delete or archive
```

### InfluxDB Performance Benchmarks

| vCPU / CPU | RAM | IOPS | Writes/sec | Queries/sec | Unique Series |
|-----------|-----|------|-----------|------------|--------------|
| 2-4 cores | 2-4 GB | 500 | < 5,000 | < 5 | < 100,000 |
| 4-6 cores | 8-32 GB | 500-1000 | < 250,000 | < 25 | < 1,000,000 |
| 8+ cores | 32+ GB | 1000+ | > 250,000 | > 25 | > 1,000,000 |

### Example: SQL vs Flux for Moving Average

**SQL (complex):**
```sql
SELECT id, temp,
  AVG(temp) OVER (PARTITION BY group_nr ORDER BY time_read) AS rolling_avg
FROM (
  SELECT id, time_read, interval_group,
    id - ROW_NUMBER() OVER (PARTITION BY interval_group ORDER BY time_read) AS group_nr
  FROM (
    SELECT time_read,
      EXTRACT(EPOCH FROM time_read)::int4 / 900 AS interval_group,
      temp
    FROM readings
  ) t1
) t2
ORDER BY time_read;
```

**Flux (simple):**
```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

---

## 3. MapReduce Aggregation: Walkthrough Example

### Example: Aggregate Ad Clicks per Ad for One Minute

**Input events (from Kafka):**

```
ad001, 2021-01-01 00:00:01, user1, 207.148.22.22, USA
ad001, 2021-01-01 00:00:02, user1, 207.148.22.22, USA
ad002, 2021-01-01 00:00:02, user2, 209.153.56.11, USA
ad001, 2021-01-01 00:00:15, user3, 192.168.1.1,   GBR
ad002, 2021-01-01 00:00:30, user4, 10.0.0.1,       GBR
ad001, 2021-01-01 00:00:45, user5, 172.16.0.1,     USA
```

**Step 1 — Map Phase:**

Map nodes partition by `ad_id % 2`:

| Map Node 0 (even ad_ids) | Map Node 1 (odd ad_ids) |
|--------------------------|------------------------|
| ad002 click (user2) | ad001 click (user1) |
| ad002 click (user4) | ad001 click (user1) |
| | ad001 click (user3) |
| | ad001 click (user5) |

**Step 2 — Aggregate Phase:**

Each aggregate node counts clicks per ad_id within the minute:

| Aggregate Node 0 | Aggregate Node 1 |
|------------------|------------------|
| ad002: count=2 | ad001: count=4 |

**Step 3 — Reduce Phase (for top N):**

Reduce node merges results:

| ad_id | click_minute | count |
|-------|-------------|-------|
| ad001 | 202101010000 | 4 |
| ad002 | 202101010000 | 2 |

Top 1 most clicked: `ad001`

### Filtered Aggregation (Star Schema)

Pre-define filter dimensions and aggregate:

| ad_id | click_minute | country | count |
|-------|-------------|---------|-------|
| ad001 | 202101010000 | USA | 3 |
| ad001 | 202101010000 | GBR | 1 |
| ad002 | 202101010000 | USA | 1 |
| ad002 | 202101010000 | GBR | 1 |

Query: "clicks for ad001 in USA" → directly look up: 3.

### Fault Tolerance with Snapshots

Aggregation happens in-memory. When a node fails, the aggregated state is lost.

**Recovery process:**
1. New aggregation node starts
2. Loads latest snapshot (contains: upstream Kafka offset, in-memory counters, top N state)
3. Replays only events after the snapshot offset from Kafka
4. Resumes normal processing

Snapshot contents:
```
{
  "upstream_offset": 6000,
  "aggregated_counts": {
    "ad001": 45,
    "ad002": 12
  },
  "top_n_state": ["ad001", "ad002", "ad005"],
  "timestamp": "2021-01-01T00:05:00Z"
}
```

---

## 4. Leaderboard: Redis Implementation Patterns

### Complete Redis Workflow

```
# Monthly leaderboard key naming
KEY = "leaderboard:{game_id}:{year}-{month}"
# e.g., leaderboard:chess:2021-02

# Player wins a match → increment score
ZINCRBY leaderboard:chess:2021-02 1 "user:mary1934"
# Returns new score (e.g., 851.0)

# Fetch top 10
ZREVRANGE leaderboard:chess:2021-02 0 9 WITHSCORES
# Returns: [("user:alice", 976), ("user:bob", 965), ...]

# Get a player's rank (0-indexed)
ZREVRANK leaderboard:chess:2021-02 "user:mary1934"
# Returns: 3 (meaning rank #4)

# Get neighbors (4 above, 4 below rank 361)
ZREVRANGE leaderboard:chess:2021-02 357 365 WITHSCORES

# Get total players
ZCARD leaderboard:chess:2021-02
# Returns: 25000000

# Get a player's score
ZSCORE leaderboard:chess:2021-02 "user:mary1934"
# Returns: 850.0
```

### Storage Estimation

| Component | Size per Entry | Entries | Total |
|-----------|---------------|---------|-------|
| User ID (24 chars) | 24 bytes | 25M MAU | 600 MB |
| Score (16-bit int) | 2 bytes | 25M MAU | 50 MB |
| Skip list + hash overhead | ~2x raw | — | ~1.3 GB |

A single Redis server (16+ GB) comfortably handles 25M users.

### Fixed Partition Sharding (for 500M DAU)

```
Score range: 1-1000
10 shards, each covers 100 points:

Shard 0: scores [1, 100]      → Redis node 0
Shard 1: scores [101, 200]    → Redis node 1
...
Shard 9: scores [901, 1000]   → Redis node 9

Secondary cache: user_id → current_score (for shard routing)
```

**Operations with sharding:**

| Operation | Algorithm |
|-----------|----------|
| **Update score** | Lookup current score in secondary cache → determine current shard → if score changes shard, `ZREM` from old + `ZADD` to new → update secondary cache |
| **Top 10** | Fetch top 10 from highest-score shard only (shard 9) |
| **User rank** | `ZREVRANK` in user's shard (local rank) + sum of `ZCARD` from all higher shards |

### Tie-Breaking Implementation

```
# Store timestamp of most recent win
HSET leaderboard:timestamps:2021-02 "user:mary1934" 1613707265

# When two users have same score:
# User with OLDER timestamp (reached score first) ranks higher
# Application-level logic:
scores = ZREVRANGE leaderboard:chess:2021-02 0 9 WITHSCORES
for users_with_same_score:
    timestamps = HMGET leaderboard:timestamps:2021-02 user1 user2
    sort by timestamp ASC (earlier = higher rank)
```

### Failure Recovery Script

```
# Rebuild leaderboard from MySQL
SELECT user_id, COUNT(*) as wins
FROM game_results
WHERE game_month = '2021-02' AND result = 'win'
GROUP BY user_id;

# For each user:
ZADD leaderboard:chess:2021-02 {wins} "user:{user_id}"
```

### DynamoDB Alternative: Write Sharding Pattern

```
Partition key: "chess#2021-02#p{user_id % N}"
Sort key: score

# N = number of partitions (e.g., 3)
# User "mary1934" → hash("mary1934") % 3 = 1 → partition p1

# Fetch top 10 (scatter-gather):
for partition in [p0, p1, p2]:
    results += Query(PK="chess#2021-02#p{partition}", ScanIndexForward=false, Limit=10)
sort(results, by=score, descending=true)
return results[:10]
```

---

## 5. Data Pipeline Technology Comparison

### Stream Processing Frameworks

| Framework | Model | Latency | Exactly-Once | State Management | Deployment |
|-----------|-------|---------|-------------|-----------------|------------|
| **Apache Flink** | True streaming (event-at-a-time) | Very low (ms) | Yes (distributed snapshots) | Built-in (RocksDB, in-memory) | Standalone, YARN, K8s |
| **Kafka Streams** | Stream processing library | Low (ms) | Yes (Kafka transactions) | Built-in (RocksDB) | Embedded in app (no cluster) |
| **Apache Spark Streaming** | Micro-batch | Medium (seconds) | Yes (with idempotent sinks) | Via DStreams/checkpointing | Standalone, YARN, K8s, Mesos |
| **Spark Structured Streaming** | Micro-batch / continuous | Low-medium | Yes (with idempotent sinks) | Via Delta Lake / checkpointing | Same as Spark |
| **Apache Storm** | True streaming | Very low (ms) | At-least once (Trident for exactly-once) | External (Redis, DB) | Standalone cluster |

### When to Choose What

| Scenario | Recommended | Why |
|----------|------------|-----|
| Low-latency event processing with complex state | **Flink** | True streaming + built-in state + event-time processing |
| Simple stream processing, already using Kafka | **Kafka Streams** | No separate cluster; library embedded in your app |
| Large-scale batch + some streaming | **Spark** | Unified batch/stream API; largest ecosystem |
| Need exactly-once with Kafka end-to-end | **Flink** or **Kafka Streams** | Both support Kafka transactions |
| Real-time aggregation with SQL-like queries | **Flink SQL** or **KSQL** | SQL interface over streams |

### Message Queue / Event Bus Comparison

| System | Ordering | Retention | Consumer Model | Throughput | Use Case |
|--------|---------|-----------|---------------|-----------|----------|
| **Kafka** | Per-partition | Configurable (days/forever) | Pull (consumer groups) | Very high (millions msg/sec) | Event streaming, data pipelines, log aggregation |
| **Pulsar** | Per-partition | Tiered storage (infinite) | Pull + Push | Very high | Multi-tenant streaming, geo-replication |
| **Kinesis** | Per-shard | 1-365 days | Pull (enhanced fan-out) | High | AWS-native streaming |
| **RabbitMQ** | Per-queue (FIFO) | Until consumed | Push | Medium | Task queues, RPC, traditional messaging |
| **SQS** | Best-effort (FIFO available) | 1-14 days | Pull | High | Simple decoupling, serverless triggers |

---

## 6. Pull vs Push Monitoring: Deep Dive

### Pull Model (Prometheus-style)

**Architecture:**
```
Service Discovery (etcd/ZK/Consul)
    ↓ (service endpoints + metadata)
Metrics Collector Pool
    ↓ (HTTP GET /metrics, periodic)
Application Servers (expose /metrics endpoint)
```

**Scaling pull collectors:**
- Use consistent hash ring: each collector handles a subset of targets
- Map each monitored server to a point on the ring by its hostname
- One collector per source server — no duplicates

**Service discovery metadata:**
```yaml
- targets: ['webserver01:9090', 'webserver02:9090']
  labels:
    env: production
    region: us-west
  scrape_interval: 15s
  scrape_timeout: 10s
```

**Advantages in detail:**
- `/metrics` endpoint doubles as debugging tool (human-readable via browser)
- Health detection is automatic: no response → server is down
- Data authenticity guaranteed (only pull from known endpoints)
- Rate controlled by the collector (no producer-side thundering herd)

### Push Model (CloudWatch / Graphite-style)

**Architecture:**
```
Application Servers
    ↓ (collection agent installed)
Agent aggregates locally (counters per minute)
    ↓ (periodic push via UDP/TCP)
Load Balancer
    ↓
Metrics Collector (auto-scaling cluster)
    ↓
Time-Series DB
```

**Agent-side pre-aggregation:**
- Simple counter: increment locally, push aggregated value every minute
- Reduces network volume by 60x (vs pushing every second)
- Risk: data loss if server terminates before next push interval (auto-scaling groups)

**Buffer strategy:**
- Agent keeps small on-disk buffer when collector rejects (backpressure)
- On recovery, resends buffered data
- Trade-off: disk buffer adds durability but complicates short-lived instance cleanup

### Decision Matrix

| Factor | Choose Pull | Choose Push |
|--------|-----------|------------|
| Network topology | Simple, flat network | Multiple DCs, firewalls, NAT |
| Workload type | Long-running services | Short-lived jobs, serverless functions |
| Control | Want collector-controlled rate | Want producer-controlled rate |
| Debugging | Need easy `/metrics` inspection | Don't need endpoint access |
| Scale pattern | Hundreds to thousands of targets | Tens of thousands of ephemeral producers |
| Existing infra | Prometheus ecosystem | AWS / managed cloud |

### Hybrid Approach

Most large organizations support both:
- **Pull** for long-running infrastructure services (known endpoints)
- **Push** for serverless functions, batch jobs, edge devices
- **Push gateway** bridges the gap: short-lived jobs push to gateway, collector pulls from gateway

```
Long-running services ←── Pull ←── Prometheus
Serverless / batch    ──→ Push ──→ Push Gateway ←── Pull ←── Prometheus
```
