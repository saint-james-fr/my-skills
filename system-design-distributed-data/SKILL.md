---
name: system-design-distributed-data
description: Distributed data patterns — CAP theorem, consistent hashing, data partitioning, replication, consistency models, conflict resolution, and failure handling. Use when designing distributed key-value stores, choosing consistency vs availability trade-offs, implementing data partitioning, or handling failures in distributed systems.
---

# Distributed Data — System Design Reference

Based on *System Design Interview* (Vol. 1, Ch. 5–6) and *DDIA* concepts.

## 1. CAP Theorem

| Property | Definition |
|----------|-----------|
| **Consistency** | Every read receives the most recent write or an error |
| **Availability** | Every request receives a (non-error) response |
| **Partition Tolerance** | System operates despite network partitions between nodes |

Network partitions are **inevitable** in distributed systems → you choose **CP or AP**.

### CP vs AP Decision Table

| Category | Sacrifices | Behaviour during partition | Real-world examples |
|----------|-----------|---------------------------|---------------------|
| **CP** | Availability | Blocks writes / returns errors until consistency restored | HBase, MongoDB (default), ZooKeeper, etcd, Redis Cluster |
| **AP** | Consistency | Accepts reads/writes; resolves conflicts later | Cassandra, DynamoDB, CouchDB, Riak |
| **CA** | Partition tolerance | **Cannot exist** in a real distributed system | Single-node RDBMS (not distributed) |

### When to choose

| Use case | Choose | Why |
|----------|--------|-----|
| Banking / financial transactions | CP | Must never show stale balance |
| Social media feeds, DNS | AP | Stale data acceptable; uptime critical |
| Shopping cart (Dynamo-style) | AP | Merge conflicts client-side; never lose writes |
| Distributed locks / leader election | CP | Correctness > availability |

### ACID vs BASE

| ACID (CP-leaning) | BASE (AP-leaning) |
|-------------------|-------------------|
| Atomicity | **Ba**sically Available |
| Consistency | **S**oft state |
| Isolation | **E**ventual consistency |
| Durability | |

## 2. Consistent Hashing

### The Rehashing Problem

With `serverIndex = hash(key) % N`, adding/removing a server remaps **almost all** keys → cache miss storm.

| Servers | key0 | key1 | key2 | key3 | key4 | key5 | key6 | key7 |
|---------|------|------|------|------|------|------|------|------|
| N=4 | s1 | s0 | s2 | s0 | s1 | s3 | s2 | s3 |
| N=3 (s1 down) | s0 | s0 | s1 | s2 | s1 | s0 | s1 | s0 |

Result: 6 of 8 keys remap — **75% miss rate**.

### Hash Ring

```
           0
          /   \
        /       \
      s0          s1
     /              \
    |    hash ring    |
     \              /
      s3          s2
        \       /
          \   /
      2^160 - 1

  key → hash(key) → walk clockwise → first server = owner
```

- Add server → only keys between new server and its predecessor remap
- Remove server → only that server's keys remap to successor

### Virtual Nodes

**Problem:** with few physical servers, partition sizes are uneven.

**Solution:** each physical server maps to `V` virtual nodes on the ring.

| Virtual nodes per server | Std deviation of load |
|--------------------------|-----------------------|
| 100 | ~10% of mean |
| 200 | ~5% of mean |
| 500+ | ~2% of mean |

Trade-off: more virtual nodes → better balance but more metadata.

### When to use consistent hashing

| Use case | Examples |
|----------|---------|
| Data partitioning | DynamoDB, Cassandra |
| Distributed caching | Memcached ring |
| Load balancing | Maglev (Google), Discord |
| CDN routing | Akamai |

## 3. Data Partitioning

### Strategies Comparison

| Strategy | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Hash-based** | `hash(key) % N` | Simple, even distribution | Massive remapping on resize |
| **Range-based** | Partition by value ranges (A-M, N-Z) | Range queries efficient | Hot spots if uneven distribution |
| **Consistent hashing** | Hash ring with virtual nodes | Minimal remapping, elastic scaling | More complex implementation |
| **Virtual bucket** | Key → virtual bucket → physical shard | Flexible rebalancing | Extra indirection layer |

### Choosing a Partition Key

| Criteria | Good key | Bad key |
|----------|----------|---------|
| Cardinality | High (user_id) | Low (country — only ~200 values) |
| Write distribution | Even across partitions | Hot key (celebrity user_id) |
| Query pattern | Matches access pattern | Requires scatter-gather |

### Rebalancing Strategies

| Strategy | How | When |
|----------|-----|------|
| Fixed partitions | Pre-allocate many partitions; assign to nodes | Elasticsearch, Riak, Kafka |
| Dynamic splitting | Split partition when it gets too large | HBase, MongoDB |
| Proportional | Number of partitions proportional to node count | Cassandra (virtual nodes) |

## 4. Data Replication

### Replication Topologies

| Topology | Write path | Consistency | Availability | Use case |
|----------|-----------|-------------|--------------|----------|
| **Single-leader** | All writes → leader → replicas | Strong (sync) or eventual (async) | Leader is SPOF without failover | Traditional RDBMS, Redis |
| **Multi-leader** | Writes to any leader → cross-replicate | Eventual; conflict resolution needed | High (multiple write points) | Multi-DC setups, CRDTs |
| **Leaderless** | Write to N nodes; read from R nodes | Tunable via quorum | Highest | DynamoDB, Cassandra, Riak |

### Synchronous vs Asynchronous Replication

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Latency | Higher (wait for replica ACK) | Lower (fire and forget) |
| Durability | Guaranteed on N nodes | Risk of data loss on leader failure |
| Availability | Lower (blocked by slow replica) | Higher |
| Consistency | Strong | Eventual |
| Example | `acks=all` in Kafka | Default MySQL replication |

### Quorum Reads/Writes

```
N = total replicas
W = write quorum (ACKs needed for success)
R = read quorum (responses needed for success)

Rule: W + R > N  →  strong consistency (overlap guarantees fresh data)
```

| Config | Optimised for | Consistency |
|--------|--------------|-------------|
| W=1, R=N | Fast writes | Strong |
| W=N, R=1 | Fast reads | Strong |
| W=2, R=2 (N=3) | Balanced | Strong |
| W=1, R=1 (N=3) | Max speed | Eventual |

### In-Sync Replicas (ISR)

ISR = set of replicas that are caught up with the leader within a configured lag.

| ACK setting | Behaviour | Trade-off |
|-------------|-----------|-----------|
| `acks=0` | No ACK; fire and forget | Lowest latency, risk of loss |
| `acks=1` | Leader ACKs | Medium latency, loss if leader dies |
| `acks=all` | All ISRs ACK | Highest durability, higher latency |

## 5. Consistency Models

| Model | Guarantee | Example |
|-------|-----------|---------|
| **Strong** | Read always returns latest write | Spanner, single-leader sync replication |
| **Eventual** | All replicas converge given enough time | DynamoDB, Cassandra, DNS |
| **Causal** | Causally related operations seen in order | MongoDB (causal sessions) |
| **Read-your-writes** | Client always sees its own writes | Session-sticky routing |
| **Monotonic reads** | Client never sees older value after newer one | Version-tracked reads |
| **Linearizable** | All ops appear atomic and in real-time order | ZooKeeper, etcd |

### Decision Guide

| Requirement | Model | Implementation cost |
|-------------|-------|-------------------|
| Banking, inventory | Strong / Linearizable | High (coordination overhead) |
| Social media, analytics | Eventual | Low |
| Collaborative editing | Causal | Medium |
| User profile after update | Read-your-writes | Low (session affinity) |

## 6. Conflict Resolution

### Vector Clocks

A `[server, version]` pair per data item to track causality.

```
D1([Sx,1])          Sx writes D1
    |
D2([Sx,2])          Sx writes D2 (descends from D1)
   / \
D3([Sx,2],[Sy,1])   Sy writes D3 (from D2)
D4([Sx,2],[Sz,1])   Sz writes D4 (from D2)
   \ /
D5([Sx,3],[Sy,1],[Sz,1])   Client merges D3+D4 → Sx writes D5
```

**Ancestor check:** X is ancestor of Y if every counter in X ≤ corresponding counter in Y.
**Conflict:** if any counter in X > corresponding counter in Y → siblings, must merge.

### Resolution Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Last-Write-Wins (LWW)** | Highest timestamp wins | Simple | Data loss (silently drops concurrent writes) |
| **Application-level merge** | Client resolves conflicts | No data loss | Complex client logic |
| **CRDTs** | Conflict-free data types that auto-merge | No coordination needed | Limited data type support |

### Anti-Entropy: Merkle Trees

Used to detect and repair inconsistencies between replicas:

```
        [Root Hash]
        /         \
   [Hash 0-1]   [Hash 2-3]
    /     \       /     \
  [H0]  [H1]  [H2]  [H3]    ← bucket hashes
  {k0}  {k1}  {k2}  {k3}    ← key buckets
```

Compare root hashes → if different, recurse down → sync only divergent buckets.
Amount of data transferred ∝ differences, not total data size.

## 7. Failure Handling

### Failure Detection: Gossip Protocol

```
Each node:
  1. Maintains membership list: [(node_id, heartbeat_counter, timestamp), ...]
  2. Periodically increments own heartbeat counter
  3. Sends heartbeats to random subset of nodes
  4. If heartbeat not updated for T seconds → mark node as down
  5. Propagation: info spreads epidemically in O(log N) rounds
```

No single point of failure for detection — fully decentralised.

### Failure Handling Strategies

| Failure type | Strategy | How it works |
|--------------|----------|-------------|
| **Temporary** | Sloppy quorum | Use first W/R *healthy* servers on ring (skip failed nodes) |
| **Temporary** | Hinted handoff | Healthy node takes over; hands data back when failed node recovers |
| **Permanent** | Anti-entropy (Merkle trees) | Compare replica hash trees; sync divergent buckets |
| **Data center outage** | Cross-DC replication | Replicas in multiple data centers; route to surviving DC |

### Sloppy Quorum + Hinted Handoff Flow

```
Normal:    Client → [s0, s1, s2]  (N=3, W=2)
s2 down:   Client → [s0, s1, s3]  (s3 temporarily holds s2's data)
s2 back:   s3 → pushes hint to s2  (s2 catches up)
```

## 8. Key-Value Store Design Summary

### System Architecture

```
  Clients
    |
    v
  [Coordinator Node]  ← any node can be coordinator (decentralised)
    |
    +-----> [Node A] ──┐
    +-----> [Node B] ──┤  Consistent hash ring
    +-----> [Node C] ──┘
            |     |
        [Memory] [SSTable on disk]

  Each node runs:
    - Client API (get/put)
    - Failure detection (gossip)
    - Replication engine
    - Conflict resolution (vector clocks)
    - Merkle tree anti-entropy
```

### Write Path

```
Request → Commit log (WAL) → Memory cache (memtable) → [flush when full] → SSTable (disk)
```

### Read Path

```
Request → Memory cache? → [hit] → return
                        → [miss] → Bloom filter → identifies candidate SSTables → read → return
```

### Techniques Summary Table

| Goal / Problem | Technique |
|----------------|-----------|
| Store big data | Consistent hashing (partition across servers) |
| High-availability reads | Data replication + multi-DC setup |
| High-availability writes | Versioning + vector clocks for conflict resolution |
| Dataset partitioning | Consistent hashing |
| Incremental scalability | Consistent hashing (virtual nodes) |
| Heterogeneous capacity | Proportional virtual nodes per server |
| Tunable consistency | Quorum consensus (W + R > N) |
| Temporary failures | Sloppy quorum + hinted handoff |
| Permanent failures | Anti-entropy with Merkle trees |
| Data center outage | Cross-data-center replication |

---

*Detailed examples → see [references/distributed-data-detail.md](references/distributed-data-detail.md)*
