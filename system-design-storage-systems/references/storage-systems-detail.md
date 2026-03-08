# Storage Systems — Detailed Reference

Extended reference for [SKILL.md](../SKILL.md).

## Erasure Coding Math and Implementation

### Reed-Solomon Fundamentals

Erasure coding uses **Reed-Solomon error correction** over Galois fields (finite fields). The core idea: given `k` data chunks, produce `m` parity chunks such that any `k` of the `k+m` total chunks can reconstruct the original data.

### (4+2) Example — Step by Step

Given data chunks `d1, d2, d3, d4`, compute parities:

```
p1 = d1 + 2·d2 - d3 + 4·d4
p2 = -d1 + 5·d2 + d3 - 3·d4
```

If `d3` and `d4` are lost, solve the system of linear equations using known values of `d1, d2, p1, p2` to reconstruct `d3` and `d4`. This works because we have 2 unknowns and 2 equations (from the 2 parity chunks).

### (8+4) Production Setup

| Parameter | Value |
|---|---|
| Data chunks (k) | 8 |
| Parity chunks (m) | 4 |
| Total chunks | 12 |
| Max tolerable failures | 4 |
| Storage multiplier | 12/8 = 1.5x |
| Durability | ~11 nines (99.999999999%) |

Each of the 12 chunks is placed on a different failure domain (rack, AZ). The encoding matrix is a Vandermonde or Cauchy matrix over GF(2^8).

### Encoding Process

1. Split object into `k` equal-sized data blocks
2. Construct a `(k+m) × k` encoding matrix
3. Multiply encoding matrix by data vector → produces `k+m` coded blocks
4. First `k` blocks = original data (systematic code)
5. Last `m` blocks = parity

### Decoding Process

1. Collect any `k` of the `k+m` blocks
2. Form a `k × k` submatrix from encoding matrix rows corresponding to available blocks
3. Invert the submatrix
4. Multiply inverted matrix by available blocks → original data

### Performance Characteristics

| Operation | Cost |
|---|---|
| Encode | O(k·m·block_size) — parallelizable across CPU cores |
| Decode (no failure) | Zero — just read the k data blocks (systematic code) |
| Decode (with failures) | O(k²·block_size) — matrix inversion + multiply |
| Read amplification | 1x (no failure) to k/available_data_blocks (with failures) |

### Implementation Libraries

- **Intel ISA-L**: Optimized erasure coding using SIMD instructions
- **Jerasure**: Reference C implementation
- **OpenEC**: Flexible erasure coding framework

## Data Placement Algorithms

### Consistent Hashing for Object Placement

The placement service maps object UUIDs to data nodes using consistent hashing:

1. Hash each data node ID onto a ring (0 to 2^32-1)
2. Hash the object UUID onto the same ring
3. Walk clockwise from the object's position to find the first `N` distinct physical nodes (N = replication factor)
4. These N nodes form the **replication group** for that object

### Virtual Nodes

Each physical data node maps to multiple positions on the ring (virtual nodes) to ensure even distribution. Typical ratio: 100-200 virtual nodes per physical node.

### Rack-Aware Placement

The placement service augments consistent hashing with topology awareness:

```
Placement constraints:
  - Primary and replicas on different racks (rack-level isolation)
  - Prefer spreading across AZs when available
  - Respect disk capacity (skip nodes above threshold)
```

### Rebalancing on Node Changes

**Node added**: Placement service assigns virtual nodes on the ring. Objects that now hash to the new node's range are migrated in background. Only neighbors are affected (minimal data movement).

**Node removed/failed**: Objects from the failed node's range are re-replicated to the next node on the ring. Placement service marks node as "down" after missing heartbeats (15s grace period).

### Virtual Cluster Map

The placement service maintains a topology map:

```
Cluster
├── AZ-1
│   ├── Rack-1
│   │   ├── Node-A (disks: 4, used: 62%)
│   │   └── Node-B (disks: 4, used: 45%)
│   └── Rack-2
│       ├── Node-C (disks: 8, used: 71%)
│       └── Node-D (disks: 8, used: 38%)
└── AZ-2
    ├── Rack-3
    │   └── Node-E (disks: 4, used: 55%)
    └── Rack-4
        └── Node-F (disks: 4, used: 29%)
```

Heartbeats update disk count and usage. Cluster map is replicated across the 5-7 node placement service cluster via consensus protocol.

## Object Versioning Implementation

### Schema Design

```sql
CREATE TABLE object (
    bucket_name  VARCHAR NOT NULL,
    object_name  VARCHAR NOT NULL,
    object_version TIMEUUID NOT NULL,  -- monotonically increasing
    object_id    UUID NOT NULL,        -- points to data store
    is_deleted   BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (bucket_name, object_name, object_version)
) WITH CLUSTERING ORDER BY (object_version DESC);
```

The clustering order ensures the latest version is always first in query results.

### Version Lifecycle

**Upload new version**:
1. Data store persists new object bytes → returns new UUID
2. Metadata store inserts new row: same `(bucket_name, object_name)`, new `object_id`, new `object_version` (TIMEUUID)
3. Old rows remain untouched

**Read current version**:
```sql
SELECT object_id FROM object
WHERE bucket_name = ? AND object_name = ?
ORDER BY object_version DESC
LIMIT 1;
```
If latest row has `is_deleted = TRUE`, return 404.

**Read specific version**:
```sql
SELECT object_id FROM object
WHERE bucket_name = ? AND object_name = ? AND object_version = ?;
```

**Delete (versioned bucket)**:
Insert a new row with `is_deleted = TRUE` (delete marker). No data is removed. All prior versions remain accessible by specifying `object_version`.

**Delete specific version**:
Remove the metadata row and mark the data object for garbage collection.

### Storage Reclamation

Old versions consume storage. Lifecycle policies can automatically:
- Delete versions older than N days
- Keep only the last N versions
- Transition old versions to cheaper storage tiers

## Google Drive Sync Protocol Detail

### Block Server Processing Pipeline

```
Input file
  → Split into blocks (max 4MB each)
    → Compute SHA-256 hash per block
      → Check hash against metadata DB (dedup)
        → If exists: skip upload, reference existing block
        → If new: compress → encrypt → upload to cloud storage
          → Record block metadata (hash, size, storage location)
```

### Delta Sync Algorithm

Based on rsync algorithm principles:

1. Client computes rolling checksums for each block of the modified file
2. Client sends checksums to block server
3. Block server compares against stored block checksums
4. Block server identifies:
   - **Unchanged blocks**: same checksum → skip
   - **Modified blocks**: different checksum → request new data
   - **New blocks**: no match → request new data
   - **Deleted blocks**: present in old, absent in new → mark for removal
5. Client uploads only changed/new blocks
6. Block server updates file version metadata with new block list

### Conflict Resolution Protocol

```
Timeline:
  t0: User A opens file (version 3)
  t1: User B opens file (version 3)
  t2: User A saves changes → version 4 created (first writer wins)
  t3: User B saves changes → CONFLICT detected

Conflict detection:
  - User B's save references base version 3
  - Server current version is 4 (modified since B's read)
  - Server rejects B's write as conflicting

Resolution:
  - Server stores B's version as a conflict copy
  - B receives both versions:
    1. Server version (A's changes, version 4)
    2. B's local version (conflict copy)
  - B manually merges or chooses one version
```

### Notification Service Protocol (Long Polling)

```
Client                          Notification Server
  |                                    |
  |--- HTTP request (long poll) ------>|
  |                                    | (holds connection open)
  |                                    | ... waits for events ...
  |                                    |
  |                                    | (file change detected)
  |<--- HTTP response (change list) ---|
  |                                    |
  |--- New HTTP request (long poll) -->|  (immediately reconnect)
  |                                    |
```

Connection timeout: server returns empty response if no events within timeout window. Client immediately re-establishes connection.

Each notification server handles ~1M+ concurrent connections. Server failure → all connected clients must reconnect to a different server (gradual reconnection to avoid thundering herd).

### File Metadata Update Sequence

On file change:
1. **Pending**: Metadata DB entry created, file status = "pending"
2. **Uploading**: Block server processing blocks
3. **Uploaded**: All blocks stored, status updated to "uploaded"
4. **Notified**: Notification service informed, other clients alerted

Each state transition is recorded. Clients that miss real-time notifications catch up via the offline backup queue.

## Block Server Chunking Algorithms

### Fixed-Size Chunking

Simplest approach: split file at fixed byte boundaries (e.g., every 4MB).

| Pros | Cons |
|---|---|
| Simple implementation | Poor dedup when data shifts (insert at start invalidates all chunks) |
| Predictable chunk sizes | Wastes bandwidth on small modifications |

### Content-Defined Chunking (CDC)

Uses a rolling hash (e.g., Rabin fingerprint) to define chunk boundaries based on content rather than position:

1. Slide a window (e.g., 48 bytes) over the file
2. Compute rolling hash at each position
3. When hash matches a pattern (e.g., last 13 bits are zero), mark as chunk boundary
4. Average chunk size ≈ 2^13 = 8KB (tunable via pattern)

| Pros | Cons |
|---|---|
| Insertions/deletions only affect nearby chunks | Variable chunk sizes |
| Excellent dedup ratio | More complex implementation |
| Minimal re-upload on edits | Slightly more CPU |

### Chunk Size Tradeoffs

| Chunk size | Dedup ratio | Metadata overhead | Transfer efficiency |
|---|---|---|---|
| Small (512B–4KB) | High | High (many entries) | Poor (many small transfers) |
| Medium (4KB–4MB) | Moderate | Moderate | Good |
| Large (4MB+) | Low | Low | Poor (re-upload large blocks for small changes) |

Dropbox uses **max 4MB** block size as a practical balance.

## Cold Storage Tiers and Lifecycle Policies

### Storage Tiers

| Tier | Access Latency | Cost (relative) | Use Case |
|---|---|---|---|
| **Standard** | Milliseconds | 1x | Frequently accessed data |
| **Infrequent Access** | Milliseconds | 0.5x | Data accessed < 1x/month |
| **Glacier / Archive** | Minutes to hours | 0.1x | Long-term archival, compliance |
| **Deep Archive** | 12-48 hours | 0.03x | Rarely accessed, regulatory retention |

### Lifecycle Policy Configuration

Typical policy chain:

```
Day 0:     Object created → Standard tier
Day 30:    No access → move to Infrequent Access
Day 90:    No access → move to Glacier
Day 365:   No access → move to Deep Archive
Day 730:   Delete (or retain per compliance)
```

### Implementation Details

- **Lifecycle daemon** scans metadata for objects matching age/access rules
- **Transition**: Metadata updated to reflect new tier; data migrated in background
- **Retrieval from cold**: Initiate restore request → object staged to Standard tier → available for download after staging delay
- **Minimum storage duration**: Most cold tiers enforce minimum billing period (e.g., 90 days for Glacier). Early deletion still incurs full period charges

### Cost Optimization Strategies

| Strategy | Savings |
|---|---|
| Auto-tiering via lifecycle policies | 50-90% for cold data |
| Delete incomplete multipart uploads (lifecycle rule) | Reclaim orphaned storage |
| Enable versioning with version expiration | Prevent unbounded version growth |
| Use Intelligent-Tiering (auto-moves based on access patterns) | Automated without policy management |
| Compress before upload | 2-10x depending on data type |

## Data Durability Calculation

### Three-Copy Replication

Given annual disk failure rate `p = 0.81%`:

```
P(all 3 copies fail) = p³ = (0.0081)³ ≈ 5.3 × 10⁻⁷
Durability = 1 - 5.3 × 10⁻⁷ ≈ 99.99995% (~6 nines)
```

This is a simplified model. Real calculations account for:
- Time to detect failure and re-replicate
- Correlated failures (same rack/power)
- Bit rot (silent data corruption)

### Erasure Coding (8+4)

With 12 chunks across 12 failure domains, data loss requires ≥5 simultaneous failures:

```
P(≥5 of 12 fail) = Σ(i=5 to 12) C(12,i) × p^i × (1-p)^(12-i)
                  ≈ 10⁻¹¹
Durability ≈ 99.999999999% (~11 nines)
```

### Durability vs Availability

| Concept | Definition | Typical target |
|---|---|---|
| **Durability** | Probability data is not lost over a year | 99.999999999% (11 nines) |
| **Availability** | Probability data is accessible at any given time | 99.99% (4 nines) |

Data can be durable but temporarily unavailable (e.g., all nodes in a region are down but data exists in another region that is reachable after failover).

## Write Path Optimization

### Batched Writes

Instead of fsyncing every object write, batch multiple objects into a single write buffer and flush periodically or when buffer is full. Amortizes disk I/O overhead.

### Per-Core Write Files

Multiple CPU cores processing parallel writes to a single read-write file creates contention. Solution: assign dedicated read-write files per core to eliminate lock contention.

```
Core 0 → /data/rw-core0.dat
Core 1 → /data/rw-core1.dat
Core 2 → /data/rw-core2.dat
...
```

### Write Amplification Mitigation

Compaction rewrites data, causing write amplification. Strategies to minimize:
- **Size-tiered compaction**: Merge files of similar size. Good for write-heavy workloads
- **Leveled compaction**: Maintain size tiers with bounded size ratios. Better read performance
- **Time-windowed compaction**: Group by time period. Good for TTL-based data
