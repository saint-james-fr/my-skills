---
name: system-design-messaging-event-streaming
description: Messaging and event streaming patterns — distributed message queues, Kafka architecture, pub/sub, event sourcing, CQRS, and delivery guarantees. Use when designing message queues, choosing between messaging models, implementing event-driven architectures, ensuring exactly-once delivery, or integrating Kafka.
---

# Messaging & Event Streaming

Based on *System Design Interview Vol. 2* (Ch. 4 — Distributed Message Queue, Ch. 12 — Digital Wallet / Event Sourcing / CQRS) and *ByteByteGo* archive.

---

## 1. Message Queue vs Event Streaming

| Dimension | Traditional MQ (RabbitMQ) | Event Streaming (Kafka, Pulsar) |
|-----------|--------------------------|--------------------------------|
| **Consumption** | Message deleted after ACK | Consumer tracks offset; messages retained |
| **Retention** | Until consumed (memory + overflow disk) | Configurable (days/weeks/forever) |
| **Replay** | Not supported | Replay from any offset |
| **Ordering** | No ordering guarantee | Per-partition ordering |
| **Delivery** | Push to consumer | Consumer pulls at own pace |
| **Throughput** | Lower (per-message routing) | Very high (sequential I/O, batching) |
| **Use cases** | Task queues, RPC, request routing | Log aggregation, CDC, event sourcing, streaming analytics |

Convergence is real: RabbitMQ added streams (append-only log, repeated consumption). Pulsar works as both MQ and streaming platform.

---

## 2. Messaging Models

### Point-to-Point vs Publish-Subscribe

| Aspect | Point-to-Point | Publish-Subscribe |
|--------|---------------|-------------------|
| Consumers | One message → one consumer | One message → all subscribers |
| Deletion | Removed after ACK | Retained by retention policy |
| Abstraction | Queue | Topic |
| Example | Task worker pool | Fan-out notifications |

### Topics, Partitions, Consumer Groups

```
Topic A
├── Partition 0  →  Consumer Group 1: Consumer-1
├── Partition 1  →  Consumer Group 1: Consumer-2
├── Partition 2  →  Consumer Group 1: Consumer-3
└── Partition 3  →  Consumer Group 1: (idle if only 3 consumers)

Consumer Group 2 (independent): reads all partitions separately
```

**Key rules:**
- One partition → one consumer within a group (guarantees order)
- One consumer can handle multiple partitions
- If consumers > partitions, excess consumers are idle
- All consumers in same group = point-to-point model
- Multiple groups on same topic = pub-sub model

### Push vs Pull

| Dimension | Push | Pull |
|-----------|------|------|
| Latency | Low — immediate delivery | Higher — polling interval |
| Backpressure | Consumer can be overwhelmed | Consumer controls rate |
| Batch-friendly | No | Yes — pull all available since last offset |
| Idle cost | None | Long-polling solves wasteful polling |
| Adopted by | RabbitMQ | Kafka, Pulsar |

Most streaming platforms use **pull** for throughput and consumer-driven flow control.

---

## 3. Kafka Architecture

### Core Components

| Component | Role |
|-----------|------|
| **Broker** | Server that stores partitions; cluster has many brokers |
| **Topic** | Logical channel for messages (named category) |
| **Partition** | Ordered, append-only log segment of a topic; unit of parallelism |
| **Producer** | Publishes messages to topics; includes routing + batching buffer |
| **Consumer** | Reads from partitions at tracked offset |
| **Consumer Group** | Set of consumers sharing partition assignments for a topic |
| **ZooKeeper / KRaft** | Coordination: leader election, metadata, broker discovery |

### How Messages Flow

```
Producer → hash(key) % numPartitions → Partition → Broker (leader)
                                                      ↓
                                              Follower replicas sync
                                                      ↓
                                              ACK returned to producer
                                                      ↓
Consumer Group → each consumer assigned subset of partitions → pull
```

### Partition Assignment & Rebalancing

Rebalancing triggers: consumer joins, leaves, crashes, or partition count changes.

**Process:**
1. Consumer finds coordinator broker via `hash(groupName)`
2. Coordinator detects membership change (heartbeat timeout or explicit leave)
3. Coordinator asks all consumers to rejoin
4. Coordinator elects group leader among consumers
5. Leader generates partition assignment plan (round-robin, range, sticky)
6. Coordinator broadcasts plan; consumers start consuming from new assignments

### Write-Ahead Log (WAL)

Messages stored as append-only log files on disk, segmented by size:

```
Partition-0/
├── segment-0   (offsets 0–999, inactive, read-only)
├── segment-1   (offsets 1000–1999, inactive)
└── segment-2   (offsets 2000–..., active, receives writes)
```

- New messages append to active segment only
- Old segments truncated based on retention policy
- Offset = position in partition (monotonically increasing)

### Why Kafka Is Fast

| Technique | How it helps |
|-----------|-------------|
| **Sequential I/O** | Append-only writes; OS heavily optimizes sequential disk access. Sequential disk can beat random memory access |
| **Zero-copy** | `sendfile()` transfers data directly from OS page cache to network card — skips application-level copy (4 copies → 2) |
| **Batching** | Producers batch in memory buffer; brokers write large chunks; consumers fetch in batches. Amortizes network + disk overhead |
| **OS page cache** | Delegates caching to OS; avoids GC overhead of in-process caches |
| **Partition parallelism** | Each partition independently readable/writable; scales horizontally |

---

## 4. Delivery Semantics

| Semantic | Producer config | Consumer behavior | Trade-off |
|----------|----------------|-------------------|-----------|
| **At-most-once** | `ack=0`, no retry | Commit offset **before** processing | Fast; may lose messages |
| **At-least-once** | `ack=1` or `ack=all`, retry on failure | Commit offset **after** processing | Safe; may duplicate |
| **Exactly-once** | Idempotent producer + transactional API | Transactional consumer (atomic commit) | Correct; higher latency + complexity |

### When to use

| Semantic | Use case |
|----------|----------|
| At-most-once | Metrics, logging (loss acceptable) |
| At-least-once | Most applications; combine with idempotent consumers |
| Exactly-once | Financial transactions, billing, payment processing |

### ACK Levels

| ACK | Behavior | Durability | Latency |
|-----|----------|------------|---------|
| `ack=0` | Fire and forget; no broker confirmation | Lowest | Lowest |
| `ack=1` | Leader persists; no follower sync wait | Medium — lost if leader dies before replication | Medium |
| `ack=all` | All ISRs must persist | Highest (bounded by `min.insync.replicas`) | Highest |

---

## 5. Replication & Durability

### In-Sync Replicas (ISR)

- ISR = set of replicas fully caught up with leader (within configured lag)
- Leader is always in ISR
- Followers removed from ISR if they fall behind `replica.lag.max.messages`
- Only ISR replicas are eligible for leader election

### Leader Election

1. Controller broker (elected via ZooKeeper/KRaft) monitors broker health
2. When leader broker fails, controller picks new leader from ISR
3. New leader starts serving reads/writes; followers re-sync

### Key Configs

| Config | Effect |
|--------|--------|
| `min.insync.replicas=2` | Writes rejected if fewer than 2 ISRs available (with `ack=all`) |
| `replication.factor=3` | 3 copies of each partition across brokers |
| `unclean.leader.election.enable=false` | Prevent out-of-sync replica from becoming leader (avoids data loss) |

### Broker Failure Recovery

1. Broker 3 crashes → partitions on it become under-replicated
2. Controller generates new replica distribution plan
3. New replicas created on healthy brokers, start syncing from leader
4. Once caught up, old references removed

**Rule:** Replicas of same partition must be on different brokers (ideally different racks/AZs).

---

## 6. Event Sourcing

### Command → Event → State Model

```
Command (intention)       Event (fact)              State (derived)
─────────────────────    ───────────────────       ─────────────────
"Transfer $1 A→C"   →   "Transferred $1 A→C"  →  A: $99, C: $101
      ↓                        ↓
 FIFO queue              Immutable append-only
 (may be invalid)         log (always valid)
```

**Key terms:**

| Term | Definition |
|------|-----------|
| **Command** | Intent from external world; may be invalid; queued FIFO |
| **Event** | Validated fact; immutable; deterministic; stored in append-only log |
| **State** | Current snapshot derived by applying events; stored in DB/KV store |
| **State Machine** | Two functions: (1) validate command → generate events, (2) apply event → update state |

### Event Store Design

Events stored in append-only log (Kafka, local WAL, or event store DB).

**Properties:**
- Immutable — events never modified or deleted
- Ordered — strict FIFO ordering preserved
- Deterministic replay — same events → same state (state machine has no randomness)

### State Reconstruction

```
Events:  [A:-$1, C:+$1, B:-$5, A:+$5, ...]
           ↓ replay from start (or snapshot)
State:   {A: $103, B: $95, C: $101, ...}
```

### Snapshots

- Periodically save full state to file/object storage (HDFS, S3)
- On restart: load latest snapshot → replay only events after snapshot
- Financial systems: daily snapshots at midnight for auditing

### When to Use / When Not to Use

| Use when | Avoid when |
|----------|-----------|
| Audit trail required (finance, compliance) | Simple CRUD with no history needs |
| Need to reconstruct any historical state | High-volume writes where event log grows unmanageably |
| Complex domain with business rule changes | Team unfamiliar with event-driven patterns |
| Debugging requires full reproducibility | Read-heavy with simple query patterns |

---

## 7. CQRS (Command-Query Responsibility Segregation)

### Core Idea

```
                    ┌─── Write Path ───┐
Commands → State Machine → Event Store → State DB
                                  ↓
                         Event published
                                  ↓
              ┌── Read Path (projections) ──┐
              Read SM 1 → Balance Query DB
              Read SM 2 → Audit Trail DB
              Read SM 3 → Analytics View
```

### Separate Command and Query Models

| Aspect | Command (write) side | Query (read) side |
|--------|---------------------|-------------------|
| Model | Normalized, event-driven | Denormalized, query-optimized |
| Store | Event log + state DB | Read-optimized views (materialized) |
| Consistency | Strongly consistent | Eventually consistent |
| Scaling | Scale writes independently | Scale reads independently with multiple projections |

### Read Model Projections

- Each read model is a **read-only state machine** consuming the event stream
- Different projections for different query needs (balance lookup, audit trail, analytics)
- Projections lag behind writes but always catch up

### Eventual Consistency

- Write side commits event → read side asynchronously rebuilds view
- Acceptable in most systems; critical reads can go to write-side DB

### When to Combine with Event Sourcing

| Combined | Standalone CQRS |
|----------|----------------|
| Need full event history + multiple read views | Different read/write models sufficient |
| Finance, compliance, audit | E-commerce catalog (separate search index from write DB) |
| Event-driven microservices | Systems where read/write asymmetry is the only concern |

---

## 8. Cloud Messaging Patterns

| Pattern | Description | Use case |
|---------|------------|----------|
| **Request/Reply (Async)** | Client sends request, gets 202 Accepted, polls for result | Long-running backend tasks |
| **Fire-and-Forget** | Producer sends message; no response expected | Logging, metrics, notifications |
| **Pub/Sub** | Publisher sends to topic; all subscribers receive | Event broadcasting, fan-out |
| **Fan-out** | Single message triggers multiple parallel consumers | Order placed → inventory + billing + email |
| **Competing Consumers** | Multiple consumers on same queue; each message processed once | Worker pool, load distribution |
| **Dead Letter Queue (DLQ)** | Failed messages moved to separate queue after max retries | Poison message isolation, debugging |
| **Priority Queue** | Messages processed by priority, not FIFO | Critical alerts before batch jobs |
| **Claim Check** | Large payload stored externally; only reference sent in message | Video processing, large file workflows |
| **Saga** | Coordinated multi-service transactions via events | Distributed transactions across microservices |

---

## 9. Scalability

### Partition-Level Parallelism

- Each partition is an independent unit of read/write
- More partitions = more parallelism (but more overhead: file handles, rebalancing time)
- Partition key determines distribution: `hash(key) % numPartitions`
- Adding partitions: new messages go to all partitions; no data migration needed
- Removing partitions: decommissioned partition serves reads until retention expires

### Consumer Group Scaling

| Action | Effect |
|--------|--------|
| Add consumer (< partition count) | Rebalance assigns more partitions per consumer → higher throughput |
| Add consumer (= partition count) | Optimal: 1 consumer per partition |
| Add consumer (> partition count) | Excess consumers idle |
| Remove consumer | Rebalance redistributes partitions to remaining consumers |

### Broker Scaling

**Adding a broker:**
1. Controller creates new replicas on new broker
2. New replicas sync from leaders (temporarily more replicas than configured)
3. Once caught up, redundant replicas on old brokers removed

**Removing a broker:**
1. Controller redistributes partitions off the departing broker
2. New replicas created elsewhere and synced
3. Broker drained and removed

### Quick Sizing Reference

| Concern | Knob |
|---------|------|
| Throughput | ↑ partitions, ↑ consumers, ↑ batch size |
| Latency | ↓ batch size, ↓ linger.ms, ack=1 |
| Durability | ↑ replication factor, ack=all, ↑ min.insync.replicas |
| Storage | Retention period, compression, log compaction |

---

## 10. Interview Checklist

When designing a messaging/streaming system:

1. **Clarify model** — MQ (task distribution) or event streaming (log-based, replay)?
2. **Delivery guarantees** — at-most/at-least/exactly-once? Idempotency on consumer side?
3. **Ordering** — global order needed or per-partition sufficient?
4. **Retention** — how long to keep messages? Replay requirements?
5. **Throughput vs latency** — batch size and ACK level trade-offs
6. **Consumer groups** — how many independent consumers? Competing or broadcasting?
7. **Replication** — replication factor, ISR, min.insync.replicas
8. **Failure handling** — DLQ, retry strategy, consumer rebalancing
9. **Event sourcing** — append-only events, state reconstruction, snapshots
10. **CQRS** — separate read/write models? Eventual consistency acceptable?

See [references/messaging-streaming-detail.md](references/messaging-streaming-detail.md) for deep dives on Kafka internals, event sourcing implementation, exactly-once with idempotency keys, and DLQ patterns.
