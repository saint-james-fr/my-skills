---
name: system-design-database-fundamentals
description: Database fundamentals for system design — SQL vs NoSQL, database types, ACID properties, indexing (B-tree, LSM), sharding strategies, replication topologies, data modeling, and locking mechanisms. Use when choosing databases, designing data models, implementing sharding, understanding indexing strategies, or evaluating consistency and isolation levels.
---

# Database Fundamentals for System Design

## 1. Database Types

| Type | Data Model | Query Language | Scaling | Consistency | Use Case | Examples |
|------|-----------|----------------|---------|-------------|----------|----------|
| **Relational (SQL)** | Tables with rows/columns, foreign keys | SQL | Vertical primarily; horizontal via sharding | Strong (ACID) | Structured data, complex joins, transactions | MySQL, PostgreSQL, CockroachDB |
| **Document** | JSON/BSON documents, nested structures | Query API / SQL-like | Horizontal (auto-sharding) | Tunable | Content management, catalogs, user profiles | MongoDB, Couchbase, Firestore |
| **Key-Value** | Simple key → value pairs | GET/PUT/DELETE | Horizontal | Tunable (AP or CP) | Session storage, caching, carts, leaderboards | Redis, DynamoDB, Memcached |
| **Wide-Column** | Row key → column families → columns | CQL / custom API | Horizontal (native) | Tunable, eventual default | Time-series, IoT, messaging at scale | Cassandra, HBase, ScyllaDB |
| **Graph** | Nodes + edges + properties | Cypher, Gremlin, SPARQL | Limited horizontal | Varies | Social networks, fraud detection, recommendations | Neo4j, Amazon Neptune, JanusGraph |
| **Time-Series** | Timestamp + tags + fields | SQL-like / Flux / PromQL | Horizontal | Eventual / tunable | Metrics, monitoring, IoT sensor data, analytics | InfluxDB, TimescaleDB, Prometheus |

### When to Pick What

- **Need joins + transactions + data integrity** → Relational
- **Flexible schema + rapid iteration** → Document
- **Ultra-low-latency lookups** → Key-Value
- **Write-heavy + time-range queries at scale** → Wide-Column
- **Relationship traversals** → Graph
- **Metrics + time-windowed aggregations** → Time-Series

## 2. SQL vs NoSQL Decision Matrix

| Dimension | SQL (Relational) | NoSQL |
|-----------|------------------|-------|
| **Schema** | Fixed schema, migrations required | Flexible / schema-on-read |
| **Scaling** | Primarily vertical; sharding is complex | Designed for horizontal scaling |
| **Consistency** | Strong (ACID by default) | Tunable (eventual → strong) |
| **Joins** | Native, efficient multi-table joins | Limited or no joins; denormalize |
| **Transactions** | Multi-row, multi-table ACID transactions | Single-document atomic; limited multi-doc |
| **Flexibility** | Schema changes need ALTER + migration | Add fields anytime, schema evolution |
| **Query Power** | Full SQL — aggregation, subqueries, CTEs | Varies — some SQL-like, some custom APIs |
| **Maturity** | 40+ years, well-understood tooling | Newer, rapidly evolving ecosystem |
| **Best For** | Financial systems, ERP, e-commerce | Real-time apps, IoT, content, big data |

**Default to SQL** unless you have a specific reason not to:
- Super-low latency requirement
- Unstructured / no relational data
- Need only serialize/deserialize (JSON, XML)
- Massive write volume at scale

## 3. ACID Properties

| Property | Definition | Violation Consequence | Enforcement |
|----------|-----------|----------------------|-------------|
| **Atomicity** | All writes in a transaction succeed or all roll back | Partial updates leave DB in inconsistent state | Transaction log (WAL), rollback on failure |
| **Consistency** | Data must satisfy all defined rules/constraints after txn | Invariants broken (e.g., negative balance) | Constraints, triggers, application logic |
| **Isolation** | Concurrent transactions don't interfere with each other | Dirty reads, phantom reads, lost updates | Isolation levels (read committed → serializable) |
| **Durability** | Committed data survives crashes | Data loss after acknowledged commit | WAL, fsync, replication to other nodes |

### ACID vs BASE

| ACID (SQL) | BASE (NoSQL) |
|------------|-------------|
| Atomicity | **B**asically **A**vailable |
| Consistency | **S**oft state |
| Isolation | **E**ventual consistency |
| Durability | — |

ACID is strict; BASE trades consistency for availability. Use ACID for financial/transactional workloads. BASE is acceptable when stale reads are tolerable (social feeds, analytics).

## 4. Indexing

### B-tree

- Balanced tree, self-balancing on insert/delete
- **Read**: O(log n) — excellent for range queries and point lookups
- **Write**: O(log n) — in-place updates, random I/O
- **Use case**: OLTP workloads, read-heavy systems
- **Used by**: PostgreSQL, MySQL (InnoDB), Oracle, SQLite

### LSM Tree (Log-Structured Merge Tree)

- Writes go to in-memory table (memtable) → flush to sorted on-disk files (SSTables) → periodic compaction
- **Read**: O(log n) but may need to check multiple levels + bloom filters
- **Write**: O(1) amortized — sequential I/O, very fast
- **Use case**: Write-heavy workloads, time-series, event logs
- **Used by**: Cassandra, RocksDB, LevelDB, HBase, ScyllaDB

### B-tree vs LSM Tree

| Dimension | B-tree | LSM Tree |
|-----------|--------|----------|
| Write speed | Moderate (random I/O) | Fast (sequential I/O) |
| Read speed | Fast (single lookup path) | Slower (multiple levels) |
| Write amplification | Lower | Higher (compaction) |
| Read amplification | Lower | Higher (bloom filters mitigate) |
| Space amplification | Moderate | Can be higher before compaction |
| Best for | Read-heavy OLTP | Write-heavy, append-only |
| Compaction needed | No | Yes (background, can cause latency spikes) |

### Inverted Index

- Maps terms → list of document IDs containing that term
- Powers full-text search engines (Elasticsearch, Lucene, Solr)
- Supports relevance scoring (TF-IDF, BM25)

### Indexing Rules of Thumb

| Do Index | Don't Index |
|----------|------------|
| Columns in WHERE, JOIN, ORDER BY | Low-cardinality columns (boolean, status with 2-3 values) |
| Foreign keys | Columns rarely used in queries |
| Columns with high selectivity | Very wide columns (large text/blob) |
| Composite index matching query patterns | Tables with very few rows |

**Cost of indexes**: Every index slows writes (must update index on INSERT/UPDATE/DELETE) and consumes storage. Index only what you query.

## 5. Data Structures Powering Databases

| Structure | Used By | Purpose | Performance |
|-----------|---------|---------|-------------|
| **Skip List** | Redis | In-memory sorted set index | O(log n) search/insert, O(1) space per pointer level |
| **Hash Index** | Most DBs, Memcached | Key → value direct lookup | O(1) average, no range queries |
| **B-tree** | PostgreSQL, MySQL, SQLite | Balanced disk-based index | O(log n) read/write, good for ranges |
| **LSM Tree + SSTable** | Cassandra, RocksDB, LevelDB | Write-optimized disk storage | O(1) write, O(log n) read with bloom filters |
| **Inverted Index** | Elasticsearch, Lucene | Full-text search | O(1) per term lookup, O(n) for boolean queries |
| **Suffix Tree** | Text search engines | String pattern matching | O(m) search where m = pattern length |
| **R-tree** | PostGIS, MongoDB (2dsphere) | Multi-dimensional / spatial index | O(log n) for nearest-neighbor, range queries |

## 6. Sharding Strategies

### Algorithms

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Hash-based** | hash(shard_key) % N → shard | Even distribution | Resharding requires data movement; no range queries |
| **Range-based** | Partition by value ranges (A-M, N-Z) | Range queries efficient | Hotspots if ranges have uneven data |
| **Consistent Hashing** | Hash ring; add/remove nodes moves minimal data | Minimal resharding | Complexity; virtual nodes needed for balance |
| **Directory-based** | Lookup table maps key → shard | Flexible, any mapping | Lookup service is single point of failure |

### Choosing a Shard Key

- **High cardinality** — many distinct values to distribute evenly
- **Even distribution** — avoid hotspots (e.g., don't shard by celebrity user_id)
- **Query isolation** — most queries should hit a single shard
- **Immutable** — changing shard key requires moving data

### Resharding Triggers

| Trigger | Approach |
|---------|----------|
| Shard capacity exhaustion | Add shards + consistent hashing to minimize data movement |
| Uneven distribution (hotspot) | Split hot shard, reassign ranges |
| Rapid growth | Pre-split with more initial shards than needed |

### Cross-Shard Challenges

- **Joins**: Not supported across shards; denormalize or use application-level joins
- **Transactions**: No native cross-shard ACID; use saga pattern or 2PC (expensive)
- **Aggregations**: Scatter-gather across shards, merge at application layer
- **Workaround**: Denormalize so queries hit a single shard/table

## 7. Replication Topologies

### Topologies

| Topology | How It Works | Consistency | Failover | Use Case |
|----------|-------------|-------------|----------|----------|
| **Single-leader** | One primary handles writes; replicas handle reads | Strong (sync) or eventual (async) | Promote replica to leader | Most OLTP systems |
| **Multi-leader** | Multiple primaries accept writes; replicate to each other | Conflict resolution needed | Each leader is independent | Multi-datacenter, offline clients |
| **Leaderless** | Any node accepts reads/writes; quorum-based | Tunable (R + W > N) | No single point of failure | Cassandra, DynamoDB, Riak |

### Sync vs Async Replication

| Type | Latency | Data Safety | Trade-off |
|------|---------|-------------|-----------|
| **Synchronous** | Higher (wait for replica ACK) | Zero data loss on failover | Write throughput limited by slowest replica |
| **Asynchronous** | Lower (fire and forget) | Possible data loss on failover | Higher throughput, replication lag |
| **Semi-synchronous** | Medium (1 sync replica + N async) | Balanced | Common production setup (MySQL, PostgreSQL) |

### Failover

| Type | Process | Risk |
|------|---------|------|
| **Automatic** | Health check → detect failure → promote replica → redirect traffic | Split-brain if detection is wrong |
| **Manual** | DBA intervenes to promote replica | Slower recovery, human error |

### Replication Lag Effects

| Problem | Cause | Mitigation |
|---------|-------|-----------|
| **Stale reads** | Read from lagging replica | Read-after-write from leader; sticky sessions |
| **Monotonic read violations** | User reads from different replicas | Route user to same replica |
| **Causal ordering** | Replica applies writes out of order | Logical clocks, causal consistency protocols |

## 8. Locking Mechanisms

### Concurrency Control Strategies

| Mechanism | How It Works | Concurrency | Overhead | Use Case |
|-----------|-------------|-------------|----------|----------|
| **Pessimistic Locking** | Lock row/table before read/write; others wait | Low — blocks concurrent access | High — lock management, deadlock risk | High contention (inventory, banking) |
| **Optimistic Locking** | Read + version number; check version on write; retry on conflict | High — no blocking | Low — version check only | Low contention (hotel reservations, CMS) |
| **Database Constraints** | DB enforces rules (CHECK, UNIQUE); rejects violating writes | High — no app-level locking | Low — DB handles validation | Simple invariants (non-negative balance) |

### Lock Granularity

| Level | Scope | Concurrency | Overhead |
|-------|-------|-------------|----------|
| **Row-level** | Single row | High | Higher memory per lock |
| **Page-level** | Disk page (group of rows) | Medium | Medium |
| **Table-level** | Entire table | Low | Low — one lock |

### Optimistic Locking Implementation

```sql
-- Read with version
SELECT *, version FROM inventory WHERE id = 42;

-- Update only if version matches
UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE id = 42 AND version = 5;
-- 0 rows affected → conflict → retry
```

### Pessimistic Locking Implementation

```sql
BEGIN;
SELECT * FROM inventory WHERE id = 42 FOR UPDATE;
-- Row is locked; other transactions wait
UPDATE inventory SET quantity = quantity - 1 WHERE id = 42;
COMMIT;
```

## 9. Data Modeling Patterns

### Normalization vs Denormalization

| Aspect | Normalized | Denormalized |
|--------|-----------|-------------|
| Data duplication | None | Intentional duplication |
| Write performance | Good (single update point) | Slower (update multiple copies) |
| Read performance | Slower (joins required) | Fast (single table/document) |
| Storage | Efficient | More storage |
| Consistency | Easy to maintain | Risk of inconsistency |
| Best for | Write-heavy, transactional | Read-heavy, analytics |

### Joins vs Embedding (Document DBs)

| Pattern | When to Use |
|---------|------------|
| **Embed** | 1:1 or 1:few relationships; data accessed together; data doesn't change often |
| **Reference (join)** | 1:many or many:many; data changes independently; unbounded arrays |

### Read-Heavy vs Write-Heavy Optimization

| Optimization | Read-Heavy | Write-Heavy |
|-------------|-----------|------------|
| Indexes | Many — speed up queries | Few — reduce write overhead |
| Denormalization | Yes — avoid joins | No — keep writes simple |
| Caching | Aggressive (Redis, CDN) | Limited; cache invalidation is hard |
| Replication | Many read replicas | Fewer replicas; focus on write throughput |
| DB engine | B-tree based (PostgreSQL, MySQL) | LSM-tree based (Cassandra, RocksDB) |

### Hot/Cold Data Separation

| Tier | Data Age | Storage | Access Pattern |
|------|----------|---------|---------------|
| **Hot** | Recent (days/weeks) | SSD, in-memory (Redis) | Frequent reads/writes |
| **Warm** | Weeks to months | Standard SSD / HDD | Occasional queries |
| **Cold** | Months to years | Object storage (S3), archive | Rare access, batch analytics |

Move cold data via TTL-based policies, background jobs, or partitioning by date.

## 10. Database Scaling Quick Reference

```
Start: single database instance
        │
        ├── Read bottleneck?
        │   ├── Yes → Add read replicas
        │   └── Still bottleneck? → Add caching layer (Redis)
        │
        ├── Write bottleneck?
        │   ├── Moderate → Vertical scaling (bigger instance)
        │   └── Severe → Horizontal sharding
        │
        ├── Data size > single node?
        │   └── Yes → Shard by key (hash or range)
        │
        ├── Need multi-region?
        │   └── Yes → Multi-leader replication
        │
        └── Need high availability?
            └── Yes → Async replication + automatic failover
```

### Scaling Decision Table

| Signal | Action | Example |
|--------|--------|---------|
| Read QPS too high | Add read replicas | 80% read workload → 3-5 replicas |
| Write QPS too high | Shard the database | Chat messages, event logs |
| Single instance storage full | Shard or archive cold data | Data > 1 TB |
| Cross-region latency | Multi-leader or leaderless replication | Global user base |
| Query latency too high | Add indexes, caching, denormalize | Dashboard queries |
| Connection exhaustion | Connection pooling (PgBouncer, ProxySQL) | Microservices → DB |

### Before You Shard

1. **Optimize queries** — indexes, query plans, denormalization
2. **Add caching** — Redis/Memcached for hot data
3. **Read replicas** — offload read traffic
4. **Vertical scaling** — bigger machine (up to hardware limits)
5. **Connection pooling** — PgBouncer, ProxySQL
6. **Archive cold data** — move old data to cheaper storage

Sharding adds complexity: cross-shard joins, distributed transactions, operational overhead. Exhaust simpler options first.
