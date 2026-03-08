# Distributed Data — Detailed Reference

## 1. Consistent Hashing Implementation

### Basic Algorithm

```
1. Hash each server to a position on a ring [0, 2^160 - 1] using SHA-1
2. Hash each key to a position on the same ring
3. Walk clockwise from the key's position → first server encountered = owner
```

### Adding a Server

```
Before:   ... ← s0 ← [keys] ← s3 ← ...
Add s4:   ... ← s0 ← [keys A] ← s4 ← [keys B] ← s3 ← ...

Only keys in [s0, s4] remap to s4. Keys in [s4, s3] stay on s3.
All other keys unaffected.
```

### Removing a Server

```
Before:   ... ← s0 ← s1 ← [keys] ← s2 ← ...
Remove s1: ... ← s0 ← [keys] ← s2 ← ...

Only s1's keys remap to s2 (next clockwise server).
```

### Virtual Nodes Implementation

Each physical server `Si` maps to multiple positions on the ring:

```
Physical server s0 → virtual nodes: s0_0, s0_1, s0_2, ..., s0_V
Physical server s1 → virtual nodes: s1_0, s1_1, s1_2, ..., s1_V
```

Lookup: `hash(key)` → walk clockwise → find virtual node `si_j` → maps to physical server `si`.

**Heterogeneous capacity:** assign more virtual nodes to more powerful servers.

```
Server A (16 CPU, 64 GB RAM) → 300 virtual nodes
Server B (4 CPU, 16 GB RAM)  → 100 virtual nodes
Server C (8 CPU, 32 GB RAM)  → 200 virtual nodes
```

### Replication with Consistent Hashing

After finding the owner node, walk clockwise and pick the next `N-1` **unique physical servers** (skip virtual nodes belonging to already-selected physical servers).

```
Ring:  ... s0_2 ... s1_0 ... s0_1 ... s2_0 ...

key hashes between s0_2 and s1_0
→ Owner: s1 (via s1_0)
→ Next unique: s2 (via s2_0) — skip s0_1 since s0 is same physical server
→ Replicas: [s1, s2, s0]  (N=3)
```

Place replicas across different data centers / racks for fault isolation.

## 2. Vector Clock Examples

### Example: No Conflict

```
Step 1: Client A writes to server Sx
        D1([Sx, 1])

Step 2: Client B reads D1, modifies, writes to Sx
        D2([Sx, 2])

D2 descends from D1 because Sx:2 >= Sx:1 → no conflict, D2 overwrites D1.
```

### Example: Conflict

```
Step 1: D1([Sx, 1])             — initial write

Step 2: D2([Sx, 2])             — update by same server

Step 3: Two concurrent writes from D2:
        Client A → Sy:  D3([Sx, 2], [Sy, 1])
        Client B → Sz:  D4([Sx, 2], [Sz, 1])

Step 4: Conflict detection
        D3 has [Sy, 1] but D4 has no Sy entry → D3 is NOT ancestor of D4
        D4 has [Sz, 1] but D3 has no Sz entry → D4 is NOT ancestor of D3
        → D3 and D4 are SIBLINGS (conflict!)

Step 5: Client reads both D3 and D4, merges them:
        D5([Sx, 3], [Sy, 1], [Sz, 1])
        Written to Sx → increments Sx counter
```

### Conflict Detection Rules

Given two versions `X` and `Y`:

| Condition | Relationship |
|-----------|-------------|
| Every counter in X ≤ corresponding counter in Y | X is **ancestor** of Y (no conflict) |
| Every counter in Y ≤ corresponding counter in X | Y is **ancestor** of X (no conflict) |
| Some counter in X > Y, some counter in Y > X | **Siblings** — conflict, must merge |

### Vector Clock Pruning

Problem: vector clocks grow unbounded as more servers participate.

Solution: set a threshold (e.g., 10 entries). When exceeded, remove oldest `[server, version]` pairs based on timestamp.

Trade-off: may lose precision for ancestor detection, but Amazon reports this has not caused problems in production (Dynamo paper).

## 3. Gossip Protocol Mechanics

### How Gossip Works

```
Every T seconds, each node:
  1. Increments its own heartbeat counter
  2. Picks a random subset of known nodes (fanout, typically 2-3)
  3. Sends its full membership list to those nodes:
     [(node_id, heartbeat_counter, last_updated_timestamp), ...]
  4. Receiving node merges the list:
     - For each entry, keep the one with the higher heartbeat counter
     - Update the timestamp to current time for updated entries
  5. If any entry's timestamp is older than T_fail → mark node as suspected down
  6. If still no update after T_cleanup → remove from list
```

### Convergence Properties

| Property | Value |
|----------|-------|
| Rounds to converge | O(log N) where N = cluster size |
| Messages per round | O(N) total across all nodes |
| Failure detection time | T_fail (configurable, typically 5-30s) |
| False positive handling | Require multiple independent reports before marking down |

### Example: 6-Node Cluster

```
Time 0:  Node A detects its heartbeat for Node C is stale
         A cannot unilaterally declare C down

Time 1:  A gossips to B and D
         B and D now also have stale info for C

Time 2:  B gossips to E, D gossips to F
         All nodes now aware of C's stale heartbeat

Time 3:  If C's heartbeat still not updated → C is marked DOWN
         If C was temporarily slow, its heartbeat would have propagated by now
```

### Advantages Over Centralised Detection

| Approach | Failure detection | Scalability | SPOF |
|----------|------------------|-------------|------|
| Centralised heartbeat monitor | Fast (direct) | Poor (monitor bottleneck) | Yes |
| All-to-all heartbeat | Fast | O(N²) messages | No |
| **Gossip** | Slightly slower (O(log N) rounds) | O(N) messages | No |

## 4. Merkle Tree for Anti-Entropy

### Building a Merkle Tree

```
Step 1: Divide key space into buckets
        Bucket 0: keys 1-3
        Bucket 1: keys 4-6
        Bucket 2: keys 7-9
        Bucket 3: keys 10-12

Step 2: Hash each key in each bucket
        Bucket 0: hash(k1), hash(k2), hash(k3)
        Bucket 1: hash(k4), hash(k5), hash(k6)
        ...

Step 3: Compute bucket hash = hash(all key hashes in bucket)
        H0 = hash(hash(k1) + hash(k2) + hash(k3))
        H1 = hash(hash(k4) + hash(k5) + hash(k6))
        H2 = hash(hash(k7) + hash(k8) + hash(k9))
        H3 = hash(hash(k10) + hash(k11) + hash(k12))

Step 4: Build tree upward
                    Root = hash(H01 + H23)
                   /                       \
            H01 = hash(H0 + H1)      H23 = hash(H2 + H3)
            /         \               /         \
          H0          H1            H2          H3
        {k1,k2,k3}  {k4,k5,k6}  {k7,k8,k9}  {k10,k11,k12}
```

### Synchronisation Algorithm

```
Replica A                          Replica B
   |                                  |
   |--- Compare Root Hashes --------->|
   |          (different!)            |
   |                                  |
   |--- Compare H01, H23 ----------->|
   |     H01 same, H23 different     |
   |                                  |
   |--- Compare H2, H3 ------------->|
   |     H2 different, H3 same       |
   |                                  |
   |--- Sync bucket 2 keys --------->|
   |     Only keys 7-9 transferred   |
   |                                  |

Result: Only divergent data is transferred.
For 1 billion keys in 1 million buckets = 1000 keys/bucket.
```

### Advantages

| Property | Benefit |
|----------|---------|
| Logarithmic comparison | O(log B) comparisons where B = number of buckets |
| Minimal data transfer | Only sync divergent buckets |
| Incremental updates | Re-hash only affected buckets when data changes |
| Independent per range | Each partition maintains its own Merkle tree |

## 5. Dynamo-Style Architecture Walkthrough

### Overview

Dynamo (Amazon's distributed key-value store) is the blueprint for Cassandra, Riak, and Voldemort. Key design principles:

| Principle | Implementation |
|-----------|---------------|
| Incremental scalability | Consistent hashing |
| Symmetry | Every node has same responsibilities (no special roles) |
| Decentralisation | No master; peer-to-peer gossip |
| Heterogeneity | Virtual nodes proportional to capacity |

### System Components

```
┌─────────────────────────────────────────────────┐
│                   Client                         │
│              get(key) / put(key, value)          │
└──────────────────────┬──────────────────────────┘
                       │
                       v
┌──────────────────────────────────────────────────┐
│             Coordinator Node                      │
│  (any node; determined by key's position on ring)│
│                                                   │
│  ┌─────────────┐  ┌──────────────┐               │
│  │ Request      │  │ Failure      │               │
│  │ Routing      │  │ Detection    │               │
│  │ (hash ring)  │  │ (gossip)     │               │
│  └─────────────┘  └──────────────┘               │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐               │
│  │ Replication  │  │ Conflict     │               │
│  │ Engine       │  │ Resolution   │               │
│  │ (quorum W/R) │  │ (vector clk) │               │
│  └─────────────┘  └──────────────┘               │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐               │
│  │ Anti-Entropy │  │ Membership   │               │
│  │ (Merkle tree)│  │ (gossip)     │               │
│  └─────────────┘  └──────────────┘               │
└──────────────────────────────────────────────────┘
```

### Write Path (Cassandra-style)

```
Client PUT(key, value)
  │
  ├─1→ Write to commit log (WAL) — sequential disk write, durable
  │
  ├─2→ Write to memtable (in-memory sorted structure)
  │
  ├─3→ Return ACK to coordinator (once W replicas acknowledge)
  │
  └─4→ [Background] When memtable exceeds threshold:
       Flush to SSTable on disk (sorted, immutable)
       Periodic compaction merges SSTables
```

### Read Path (Cassandra-style)

```
Client GET(key)
  │
  ├─1→ Check memtable (in-memory)
  │     └─ Hit? Return immediately
  │
  ├─2→ Check Bloom filter
  │     └─ Probabilistic: tells which SSTables MIGHT contain key
  │        (false positives possible, false negatives impossible)
  │
  ├─3→ Read candidate SSTables
  │     └─ Check in reverse chronological order (newest first)
  │
  ├─4→ Merge results, return latest version
  │
  └─5→ [Background] Read repair: if R replicas returned different
       versions, push latest to stale replicas
```

### Conflict Handling During Reads

```
Coordinator sends GET to N replicas, waits for R responses:

Case 1: All R responses have same version
  → Return value, done

Case 2: Responses have different versions
  → Check vector clocks for ancestry
  → If one is ancestor of another → return latest, repair stale replica
  → If siblings (true conflict) → return all versions to client for merge
```

### Put It All Together: Full Request Lifecycle

```
1. Client calls put(key="cart-123", value={items: [...]})
2. Any node receiving request becomes coordinator
3. Coordinator hashes "cart-123" → position on ring
4. Walks clockwise to find N=3 responsible nodes: [A, B, C]
5. Sends write to A, B, C in parallel
6. Waits for W=2 ACKs
7. If B is down: sloppy quorum → sends to D instead (hinted handoff)
8. Returns success to client
9. Later: D detects B is back → pushes hinted data to B
10. Background: Merkle tree comparison catches any remaining drift
```

### Tuning Guide

| Parameter | Low value | High value |
|-----------|-----------|------------|
| N (replicas) | Lower durability, lower cost | Higher durability, higher cost |
| W (write quorum) | Faster writes, risk of inconsistency | Slower writes, stronger consistency |
| R (read quorum) | Faster reads, may read stale data | Slower reads, guaranteed fresh data |
| Virtual nodes | Less balanced load | More balanced, more memory for metadata |
| Gossip interval | Slower failure detection | Faster detection, more network traffic |
| Merkle tree granularity | Fewer buckets → coarser sync | More buckets → finer sync, more memory |

### Common Configurations

| Config | N | W | R | W+R>N? | Use case |
|--------|---|---|---|--------|----------|
| Strong consistency | 3 | 2 | 2 | Yes (4>3) | Financial data |
| Fast writes | 3 | 1 | 3 | Yes (4>3) | Write-heavy logging |
| Fast reads | 3 | 3 | 1 | Yes (4>3) | Read-heavy cache |
| Eventual consistency | 3 | 1 | 1 | No (2<3) | Best-effort, max speed |

---

*Back to main reference → see [../SKILL.md](../SKILL.md)*
