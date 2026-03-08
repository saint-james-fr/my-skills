# Database Fundamentals — Detailed Reference

## PostgreSQL Scaling: Figma 100x Case Study

Figma scaled PostgreSQL ~100x over several years without migrating away. Key strategies:

### Vertical Scaling Phase
- Started with a single PostgreSQL instance
- Upgraded to increasingly powerful machines (more CPU, RAM, IOPS)
- Effective until hardware limits are reached

### Read Replica Phase
- Added read replicas for read-heavy workloads (UI rendering, search, analytics)
- Application-level routing: writes → primary, reads → replicas
- Challenge: replication lag causes stale reads for recently-written data
- Solution: read-after-write consistency by routing recent writes' reads to primary

### Connection Pooling
- PgBouncer in front of PostgreSQL to multiplex thousands of application connections into fewer DB connections
- Without pooling: each microservice instance opens connections → connection exhaustion
- Transaction-level pooling: connection returned to pool after each transaction

### Partitioning (Table-Level Sharding)
- PostgreSQL native partitioning by range (e.g., created_at) or hash (e.g., team_id)
- Large tables split into smaller partitions → faster queries, easier maintenance
- Partition pruning: query planner skips irrelevant partitions

### Horizontal Sharding (Application-Level)
- Shard by tenant (team_id) — most queries are scoped to a single team
- Application routing layer determines which shard to query
- Cross-shard queries handled at application layer (rare for Figma's workload)

### Key Takeaways
1. PostgreSQL can scale further than most people think
2. Exhaust vertical scaling + read replicas + connection pooling before sharding
3. Choose shard key that matches access patterns (team_id for SaaS)
4. Application-level sharding gives full control but adds complexity

## Discord: Trillions of Messages (Cassandra → ScyllaDB)

### The Problem
- Discord stores trillions of messages
- Originally used Cassandra — worked at first but degraded at scale
- Cassandra's JVM garbage collection caused unpredictable latency spikes
- Compaction storms: background compaction of SSTables caused read latency to spike
- Hot partitions: popular servers (millions of members) created massive partitions

### Why Cassandra Struggled
| Issue | Impact |
|-------|--------|
| JVM GC pauses | p99 latency spikes of 5-10 seconds |
| Compaction storms | Read latency degradation during compaction |
| Large partitions | Single channel with millions of messages → slow reads |
| Repair complexity | Anti-entropy repair across cluster was expensive |

### Migration to ScyllaDB
- ScyllaDB: Cassandra-compatible (same CQL, same data model) but written in C++ (no JVM/GC)
- Shard-per-core architecture: each CPU core owns its data, no cross-core contention
- Predictable latency: no GC pauses, more efficient compaction

### Data Model Changes
- Partition key: `(channel_id, bucket)` where bucket = time window (e.g., 10 days)
- Prevents unbounded partition growth for popular channels
- Clustering key: `message_id` (Snowflake ID = timestamp + sequence)
- Range queries within a time bucket are efficient

### Results After Migration
| Metric | Cassandra | ScyllaDB |
|--------|-----------|----------|
| p99 read latency | 40-125 ms | 15 ms |
| p99 write latency | 5-70 ms | 5 ms |
| Nodes required | 177 | 72 |
| GC pauses | Frequent | None (no JVM) |

### Key Takeaways
1. JVM-based databases (Cassandra, HBase) can struggle at extreme scale due to GC
2. Data model design (bounded partitions) matters as much as database choice
3. Cassandra-compatible alternatives (ScyllaDB) allow migration without rewriting queries
4. Shard-per-core architecture eliminates cross-thread contention

## B-tree vs LSM Tree Internals

### B-tree Internal Structure

```
                    [30 | 60]              ← root (internal node)
                   /    |    \
          [10|20]    [40|50]    [70|80]     ← internal nodes
         / |  \     / |  \     / |  \
       [1-9][11-19][21-29]...              ← leaf nodes (actual data/pointers)
```

**Properties**:
- Self-balancing: all leaf nodes at same depth
- Each node = disk page (typically 4-16 KB)
- Branching factor: number of children per node (typically 100-500 for disk-based)
- Tree depth: log_b(N) where b = branching factor — typically 3-4 levels for millions of rows
- Updates are **in-place**: find the page, modify it, write it back

**Write path**:
1. Traverse tree to find correct leaf page
2. If page has space → insert in sorted order
3. If page full → split page into two, update parent
4. Write-ahead log (WAL) for crash recovery

**Read path**:
1. Start at root, binary search within node
2. Follow pointer to child node
3. Repeat until leaf node found
4. Typically 3-4 disk reads for millions of keys

### LSM Tree Internal Structure

```
Write path:
  Client write → Memtable (in-memory, sorted)
                      │
                      ▼ (flush when full)
                 Level 0: SSTable files (unsorted between files)
                      │
                      ▼ (compaction: merge + sort)
                 Level 1: SSTable files (sorted, non-overlapping)
                      │
                      ▼ (compaction)
                 Level 2: SSTable files (larger, sorted)
                      │
                      ▼
                 Level N: ...
```

**Components**:
- **Memtable**: In-memory sorted structure (red-black tree or skip list)
- **SSTable** (Sorted String Table): Immutable on-disk file of sorted key-value pairs
- **Bloom filter**: Per-SSTable probabilistic filter — quickly answers "is this key possibly here?"
- **Sparse index**: Sampled keys pointing into SSTable for binary search

**Write path**:
1. Write to WAL (for durability)
2. Insert into memtable (in-memory)
3. When memtable reaches threshold → flush to disk as new SSTable (Level 0)
4. Background compaction merges SSTables, removes duplicates/tombstones

**Read path**:
1. Check memtable first
2. Check bloom filter for each SSTable level (skip if key definitely not present)
3. Search SSTables from newest to oldest
4. Return first match found

**Compaction strategies**:
- **Size-tiered** (Cassandra default): Merge SSTables of similar size → good write throughput
- **Leveled** (RocksDB default): Each level has non-overlapping SSTables → better read performance
- **FIFO**: Drop oldest SSTables → for time-series with TTL

### Performance Comparison Deep Dive

| Metric | B-tree | LSM Tree |
|--------|--------|----------|
| Random write | ~1,000-10,000 ops/sec | ~100,000+ ops/sec |
| Random read | ~10,000-50,000 ops/sec | ~5,000-30,000 ops/sec |
| Sequential write | Similar | Significantly faster |
| Range scan | Excellent (sorted pages) | Good (sorted SSTables) |
| Space usage | ~60-70% page fill factor | More compact after compaction |
| Write amplification | 1x (in-place) | 10-30x (compaction rewrites) |
| SSD friendly | Moderate (random I/O) | Yes (sequential I/O) |

## Database Isolation Levels

### The Four Standard Levels (weakest → strongest)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|--------------|-------------|
| **Read Uncommitted** | Possible | Possible | Possible | Fastest |
| **Read Committed** | Prevented | Possible | Possible | Fast |
| **Repeatable Read** | Prevented | Prevented | Possible | Moderate |
| **Serializable** | Prevented | Prevented | Prevented | Slowest |

### Anomalies Explained

**Dirty Read**: Transaction reads data written by another uncommitted transaction.
```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;
T2: SELECT balance FROM accounts WHERE id = 1;  -- reads 0
T1: ROLLBACK;  -- balance should still be 100
-- T2 read a value that never actually existed
```

**Non-Repeatable Read**: Same query in one transaction returns different values.
```
T1: SELECT balance FROM accounts WHERE id = 1;  -- returns 100
T2: UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  -- returns 200 (changed!)
```

**Phantom Read**: Same query returns different rows (new rows appear).
```
T1: SELECT * FROM orders WHERE amount > 100;  -- returns 5 rows
T2: INSERT INTO orders (amount) VALUES (150); COMMIT;
T1: SELECT * FROM orders WHERE amount > 100;  -- returns 6 rows (phantom)
```

### Implementation Mechanisms

| Level | PostgreSQL | MySQL (InnoDB) |
|-------|-----------|---------------|
| Read Committed | MVCC — each statement sees latest committed snapshot | MVCC — consistent read per statement |
| Repeatable Read | MVCC — snapshot at transaction start | MVCC + gap locks (prevents phantoms too) |
| Serializable | SSI (Serializable Snapshot Isolation) — detects conflicts | Traditional 2PL (two-phase locking) |

### Practical Defaults

| Database | Default Level |
|----------|--------------|
| PostgreSQL | Read Committed |
| MySQL (InnoDB) | Repeatable Read |
| Oracle | Read Committed |
| SQL Server | Read Committed |
| CockroachDB | Serializable |

**Rule of thumb**: Use your database's default unless you have a specific reason to change it. Serializable is safest but slowest — use for financial transactions, double-booking prevention.

## Connection Pooling and Query Optimization

### Connection Pooling

**Problem**: Each database connection consumes ~5-10 MB of memory. A microservices architecture with 50 services × 20 instances × 10 connections = 10,000 connections → exceeds DB limits.

**Solution**: Connection pooler sits between app and DB.

```
App instances (1000s of connections)
        │
        ▼
   PgBouncer / ProxySQL (pool of ~100 connections)
        │
        ▼
   PostgreSQL (handles 100 real connections)
```

| Pooler | DB Support | Pooling Modes |
|--------|-----------|---------------|
| **PgBouncer** | PostgreSQL only | Session, Transaction, Statement |
| **ProxySQL** | MySQL | Connection multiplexing, query routing |
| **Pgpool-II** | PostgreSQL | Pooling + load balancing + replication |
| **Application-level** | Any (HikariCP, c3p0) | Configurable per app |

**Pooling modes (PgBouncer)**:
- **Session**: Connection held for entire session (least efficient)
- **Transaction**: Connection returned after each transaction (recommended)
- **Statement**: Connection returned after each statement (most restrictive — no multi-statement txns)

### Query Optimization Checklist

| Step | Action | Tool |
|------|--------|------|
| 1 | Identify slow queries | `pg_stat_statements`, slow query log |
| 2 | Analyze query plan | `EXPLAIN ANALYZE` |
| 3 | Check index usage | `EXPLAIN` → Seq Scan = missing index |
| 4 | Add covering indexes | `CREATE INDEX ... INCLUDE (col)` |
| 5 | Rewrite N+1 queries | Use JOINs or batch loading |
| 6 | Avoid SELECT * | Select only needed columns |
| 7 | Use pagination properly | Keyset pagination > OFFSET |
| 8 | Tune work_mem, shared_buffers | PostgreSQL config |

### Common Query Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `SELECT *` | Fetches unused columns, can't use covering index | List specific columns |
| `OFFSET 10000` | DB scans and discards 10,000 rows | Keyset pagination: `WHERE id > last_seen_id LIMIT 20` |
| N+1 queries | 1 query for list + N queries for details | JOIN or batch `WHERE id IN (...)` |
| Missing index on JOIN column | Full table scan on join | Add index on foreign key |
| `LIKE '%term%'` | Can't use B-tree index | Full-text search index or trigram index |
| Functions on indexed columns | `WHERE YEAR(created_at) = 2024` bypasses index | `WHERE created_at >= '2024-01-01'` |

## Top 10 Open-Source Databases Comparison

| Database | Type | Language | Scaling | Best For | License |
|----------|------|----------|---------|----------|---------|
| **MySQL** | Relational | C/C++ | Replication, sharding (Vitess) | Web apps, WordPress, SaaS | GPL v2 |
| **PostgreSQL** | Relational | C | Replication, partitioning, Citus | Complex queries, GIS, JSON | PostgreSQL License |
| **MariaDB** | Relational | C/C++ | Replication, Galera Cluster | MySQL drop-in replacement | GPL v2 |
| **Cassandra** | Wide-column | Java | Native horizontal (consistent hashing) | Write-heavy, time-series, IoT | Apache 2.0 |
| **MongoDB** | Document | C++ | Native sharding, replica sets | Flexible schema, rapid dev | SSPL |
| **Redis** | Key-value | C | Redis Cluster (hash slots) | Caching, sessions, pub/sub | BSD (≤7.2), dual (7.4+) |
| **SQLite** | Relational (embedded) | C | Single-file, no server | Mobile, embedded, testing | Public Domain |
| **CockroachDB** | Distributed SQL | Go | Native horizontal, multi-region | Global ACID transactions | BSL / Apache |
| **Neo4j** | Graph | Java | Causal clustering | Relationship-heavy data | GPL / Commercial |
| **Couchbase** | Document + KV | C++ | Auto-sharding, cross-datacenter | Mobile sync, caching + persistence | Apache 2.0 |

### Selection Heuristics

```
Need ACID + complex queries?
  ├── Single region → PostgreSQL
  └── Multi-region → CockroachDB

Need write-heavy at scale?
  ├── Time-series / IoT → Cassandra or ScyllaDB
  └── Flexible schema → MongoDB

Need sub-millisecond latency?
  ├── Cache layer → Redis
  └── Embedded / edge → SQLite

Need relationship traversal?
  └── Graph queries → Neo4j

Need full-text search?
  └── Elasticsearch (not a primary DB)
```

## Real-World Database Choices by Company

| Company | Database(s) | Workload | Scale |
|---------|-----------|----------|-------|
| **Figma** | PostgreSQL (sharded by team) | Collaborative design tool | Hundreds of millions of files |
| **Discord** | ScyllaDB (migrated from Cassandra) | Chat messages | Trillions of messages |
| **Netflix** | Cassandra, CockroachDB, MySQL | User data, viewing history | 200M+ subscribers |
| **Uber** | MySQL (Docstore), Cassandra, Redis | Trips, maps, real-time | Millions of trips/day |
| **Slack** | MySQL (Vitess), Redis | Messaging, presence | Billions of messages |
| **Meta** | MySQL (MyRocks), TAO (graph), ZippyDB | Social graph, messages | Billions of users |
| **Google** | Spanner, Bigtable, Firestore | Global transactions, analytics | Planet-scale |
| **Airbnb** | MySQL, Redis, Elasticsearch | Listings, search, bookings | Millions of listings |
| **Stripe** | PostgreSQL (heavily tuned) | Payment processing | Billions of API calls/year |
| **Shopify** | MySQL (Vitess), Redis | E-commerce | Millions of merchants |
