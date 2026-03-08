---
name: system-design-storage-systems
description: Storage system patterns — block, file, and object storage, S3-like object storage design, file sync systems (Google Drive), erasure coding, deduplication, and garbage collection. Use when designing storage services, choosing between storage types, implementing file sync, or ensuring data durability.
---

# Storage Systems Design Reference

Based on *System Design Interview* Vol 1, Chapter 15 (Google Drive) and Vol 2, Chapter 9 (S3-like Object Storage).

## Storage Types Comparison

| Aspect | Block Storage | File Storage | Object Storage |
|---|---|---|---|
| **Abstraction** | Raw blocks (volume) | Files + directories | Flat key-value (URI → blob) |
| **Access** | SAS/iSCSI/Fibre Channel | SMB/CIFS, NFS | RESTful API |
| **Performance** | Medium-high to very high | Medium-high | Low-medium |
| **Scalability** | Medium | High | Vast |
| **Mutability** | Yes | Yes | No (versioning, not in-place update) |
| **Cost** | High | Medium-high | Low |
| **Consistency** | Strong | Strong | Strong (e.g. S3) |
| **Use cases** | VMs, databases, low-latency apps | General-purpose file sharing | Archival, backup, unstructured data |
| **Examples** | EBS, local SSD/HDD | EFS, NFS shares | S3, GCS, Azure Blob |

**Decision rule**: Need raw performance → block. Need shared file hierarchy → file. Need vast scale + durability at low cost → object.

## S3-like Object Storage Design

### Design Philosophy

Object storage separates **metadata** from **data**, like UNIX separates inodes from file blocks. Metadata store holds mutable object info; data store holds immutable object bytes. Optimize each independently.

### High-Level Architecture

```
Client → Load Balancer → API Service → IAM (auth/authz)
                              ├──→ Metadata Store (buckets, objects, versions)
                              └──→ Data Store (actual bytes, UUID-based)
```

| Component | Role |
|---|---|
| **API service** | Stateless orchestrator; routes to IAM, metadata, data store |
| **IAM** | Authentication + authorization + access control |
| **Data store** | Persists/retrieves object bytes by UUID |
| **Metadata store** | Stores bucket and object metadata (name → UUID mapping) |

### Upload Flow

1. Client sends `PUT /bucket/object` to API service
2. API service verifies identity + WRITE permission via IAM
3. API service sends payload to data store → data store returns UUID
4. API service creates metadata entry: `(bucket_id, object_name, object_id=UUID)`

### Download Flow

1. Client sends `GET /bucket/object` to API service
2. API service verifies READ permission via IAM
3. API service queries metadata store: `object_name → UUID`
4. API service fetches bytes from data store by UUID
5. Returns object data to client

### Data Store Internals

```
API Service → Data Routing Service → Placement Service (virtual cluster map)
                                   → Data Nodes (actual disk storage)
```

| Component | Responsibility |
|---|---|
| **Data routing service** | Stateless; queries placement, routes reads/writes to data nodes |
| **Placement service** | Maintains virtual cluster map; assigns replicas across failure domains; 5-7 node Paxos/Raft cluster |
| **Data nodes** | Store objects on disk; send heartbeats to placement service |

**Data persistence flow**:
1. API service forwards object to data routing service
2. Data routing generates UUID, queries placement service for target nodes
3. Routes data to primary data node
4. Primary saves locally + replicates to 2 secondaries
5. Returns UUID after all replicas confirm (strong consistency)

### Small Object Optimization

Storing each small object as a separate file wastes disk blocks and exhausts inodes. Solution: **append objects to a shared read-write file** (WAL-like). When file reaches capacity (~GBs), mark as read-only, create new file.

**Object lookup** requires: `file_name`, `start_offset`, `object_size` — stored in a local SQLite per data node.

| Field | Description |
|---|---|
| `object_id` | UUID |
| `file_name` | Data file containing the object |
| `start_offset` | Byte offset within file |
| `object_size` | Size in bytes |

### Metadata Model

**Bucket table**: `bucket_name`, `bucket_id`, `owner_id`, `enable_versioning`
**Object table**: `bucket_name`, `object_name`, `object_version`, `object_id`

**Scaling**: Bucket table fits on single DB (replicate for read throughput). Object table sharded by `hash(bucket_name, object_name)` — supports name-based lookups directly.

### Object Versioning

When versioning is enabled, uploading overwrites don't replace the metadata row — a new row is inserted with the same `(bucket_id, object_name)` but a new `object_id` and `object_version` (TIMEUUID). Current version = largest TIMEUUID.

**Deletion with versioning**: Insert a **delete marker** as the latest version. GET returns 404. Old versions remain recoverable.

### Listing Optimization

Listing with sharded metadata is expensive (scatter-gather across all shards). Solution: **denormalize listing data** into a separate table sharded by `bucket_id`. Accepts sub-optimal perf as a tradeoff — all commercial object stores do this.

## Durability & Reliability

### Failure Domains

| Level | Scope | Examples |
|---|---|---|
| **Disk** | Single drive failure | HDD/SSD dies |
| **Node** | Motherboard, PSU, all disks | Server crash |
| **Rack** | Shared switch/power | Switch failure, power loss |
| **Datacenter/AZ** | Entire facility | Natural disaster, cooling failure |

Replicate data across failure domains. Cross-AZ replication protects against large-scale failures.

### Replication vs Erasure Coding

| Aspect | 3-Copy Replication | Erasure Coding (8+4) |
|---|---|---|
| **Durability** | ~6 nines | ~11 nines |
| **Storage overhead** | 200% (3x total) | 50% (1.5x total) |
| **Write performance** | Fast (no computation) | Slower (parity calculation) |
| **Read performance** | Fast (any single replica) | Slower (read from ≥8 nodes) |
| **Read under failure** | No impact (use other replica) | Must reconstruct missing chunks |
| **Compute cost** | None | Higher (parity math) |
| **Complexity** | Simple | Complex |

**Erasure coding example (8+4)**:
- Split data into 8 equal chunks, compute 4 parity chunks
- Distribute 12 chunks across 12 failure domains
- Can tolerate up to 4 simultaneous node failures
- Storage = 12/8 = 1.5x (vs 3x for replication)

**When to use**: Replication for latency-sensitive hot data. Erasure coding for cold/archival data where storage cost dominates.

### Data Integrity — Checksums

Append checksum (MD5/SHA1) to each object. On read:
1. Fetch object data + stored checksum
2. Recompute checksum
3. Match → data is intact. Mismatch → read from another replica/reconstruct

Also append file-level checksum when marking a data file as read-only.

### Crash Recovery — Write-Ahead Log

Data files use WAL-like append-only structure. Objects are appended sequentially to a read-write file. On crash, replay from last known good offset. Read-only files are immutable and safe.

## File Sync System Design (Google Drive)

### Requirements

- Upload, download, sync files across devices
- File revisions history
- Share files, send notifications on changes
- 50M users, 10M DAU, 10GB free per user, avg 500KB file, ~240 QPS upload

### High-Level Components

| Component | Role |
|---|---|
| **Block servers** | Split files into blocks, compress, encrypt, upload to cloud storage |
| **Cloud storage** | Stores encrypted file blocks (e.g. S3, cross-region replicated) |
| **Cold storage** | Inactive data (e.g. S3 Glacier) |
| **Metadata DB** | Users, files, blocks, versions (relational, ACID) |
| **Metadata cache** | Fast retrieval of hot metadata |
| **Notification service** | Long-polling to push file change events to clients |
| **Offline backup queue** | Queues changes for offline clients |
| **API servers** | Auth, profiles, metadata updates |

### Upload Flow

Two parallel paths when client uploads:

**Path 1 — Metadata**:
1. Client sends file metadata to API server
2. Metadata DB creates entry with status = "pending"
3. Notification service alerts other clients: "file uploading"

**Path 2 — File data**:
1. Client sends file to block servers
2. Block servers chunk → compress → encrypt → upload to cloud storage
3. Cloud storage triggers completion callback
4. Metadata DB updates status to "uploaded"
5. Notification service alerts other clients: "file ready"

### Download Flow

Triggered when notification service informs client of remote change:

1. Notification service → client: "file changed"
2. Client requests updated metadata from API server
3. API server fetches metadata from DB
4. Client downloads new/changed blocks from block servers
5. Block servers fetch from cloud storage → return to client
6. Client reconstructs file from blocks

### Sync Conflict Resolution

**Strategy: first-to-save wins.**

When two users edit the same file simultaneously:
- First version processed → saved normally
- Second version → conflict detected → system presents both copies
- User resolves: merge or override

### Block-Level Chunking Benefits

| Benefit | Description |
|---|---|
| **Delta sync** | Only modified blocks transferred, not entire file |
| **Deduplication** | Identical blocks (same hash) stored once |
| **Compression** | Per-block compression (gzip for text, format-specific for media) |
| **Bandwidth savings** | Dramatically reduces network transfer for large files |

Max block size: ~4MB (Dropbox reference).

### Notification Service

Uses **long polling** (not WebSocket) because:
- Communication is unidirectional (server → client)
- Notifications are infrequent, no burst patterns
- Each server handles ~1M+ connections

### Metadata Schema (Google Drive)

| Table | Key Fields |
|---|---|
| **User** | username, email, profile |
| **Device** | device_id, push_id, user_id |
| **Namespace** | root directory per user |
| **File** | file_name, namespace_id, latest version |
| **File_version** | file_id, version_number, blocks (read-only rows) |
| **Block** | block_id, hash, file_version_id, order |

Strong consistency required → relational DB with ACID. Cache invalidation on writes.

## Storage Space Optimization

| Technique | How it works |
|---|---|
| **Block-level dedup** | Two blocks with same hash → store once. Dedup at account level |
| **Version limits** | Cap stored versions; weight toward recent versions |
| **Cold storage tiering** | Move inactive data to cheaper tier (e.g. S3 → S3 Glacier) |
| **Compression** | Per-block: gzip/bzip2 for text, format-specific for media |
| **Intelligent backup** | Full backup + incremental. Limit frequent-save versions |

## Large File Handling

### Multipart Upload

For files > few GBs, slice into parts and upload independently:

1. Client initiates multipart upload → data store returns `uploadID`
2. Client splits file into parts (e.g. 200MB each)
3. Each part uploaded with `uploadID` → data store returns `ETag` (MD5 checksum)
4. Client sends completion request with `uploadID` + part numbers + ETags
5. Data store reassembles object from parts
6. Garbage collector cleans up obsolete parts

### Resumable Uploads

1. Client sends initial request → receives resumable URL
2. Upload data, monitor progress
3. On interruption → resume from last successful byte offset

### Presigned URLs

For direct client-to-storage uploads, bypassing block servers:
- Server generates time-limited signed URL
- Client uploads directly to cloud storage
- **Tradeoff**: faster upload but chunking/encryption must happen client-side (error-prone, less secure)

## Garbage Collection

| Type | What it cleans |
|---|---|
| **Compaction** | Copies live objects from read-only files to new files, skipping deleted objects. Updates `object_mapping` table in a transaction |
| **Orphan cleanup** | Removes half-uploaded data, abandoned multipart uploads |
| **Corrupted data cleanup** | Removes objects that fail checksum verification |

**Compaction process**:
1. Wait until enough read-only files accumulate
2. Copy live objects (delete flag = false) to new consolidated files
3. Update `object_mapping` entries (file_name, start_offset) in DB transaction
4. Delete old files
5. For replication: delete from primary + all replicas. For erasure coding (8+4): delete from all 12 nodes

## Scalability Patterns

| Area | Strategy |
|---|---|
| **API service** | Stateless → horizontal scale behind load balancer |
| **Data store** | Add data nodes; placement service auto-discovers via heartbeats |
| **Metadata — bucket table** | Small enough for single DB; add read replicas |
| **Metadata — object table** | Shard by `hash(bucket_name, object_name)` |
| **Listing** | Denormalize into separate table sharded by `bucket_id` |
| **Placement service** | 5-7 node Paxos/Raft cluster |
| **Block servers** | Stateless workers; scale horizontally |
| **Notification service** | Each server handles ~1M connections; horizontal scale |

### Failure Handling Quick Reference

| Component | Recovery |
|---|---|
| Load balancer | Secondary takes over via heartbeat monitoring |
| Block server | Other servers pick up pending jobs |
| Cloud storage | Cross-region replication; fetch from other region |
| API server | Stateless; LB redirects traffic |
| Metadata cache | Replicated; replace failed node |
| Metadata DB master | Promote slave to master |
| Metadata DB slave | Use other slave; spin up replacement |
| Notification server | Clients reconnect to different server (slow reconnect) |
| Offline backup queue | Replicated; consumers re-subscribe to backup |

## Key Numbers

| Metric | Value |
|---|---|
| S3 objects stored (2021) | 100+ trillion |
| S3 durability SLA | 99.999999999% (11 nines) |
| 3-copy replication durability | ~99.9999% (6 nines) |
| Erasure coding (8+4) durability | ~99.999999999% (11 nines) |
| Replication storage overhead | 200% |
| Erasure coding (8+4) overhead | 50% |
| HDD annual failure rate | ~0.81% |
| Dropbox max block size | 4MB |
| Typical SATA HDD IOPS | 100-150 |
| Google Drive: DAU | 10M |
| Google Drive: upload QPS | ~240 (peak ~480) |

## Detailed Reference

For erasure coding math, data placement algorithms, versioning implementation, Google Drive sync protocol, block chunking, and cold storage lifecycle policies, see [storage-systems-detail.md](references/storage-systems-detail.md).
