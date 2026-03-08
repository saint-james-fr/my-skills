# Messaging & Event Streaming — Detailed Reference

Deep dives supplementing the main SKILL.md.

---

## 1. Kafka Deep Dive

### ZooKeeper Role (Legacy) → KRaft

**ZooKeeper responsibilities:**
- Broker registration and discovery (which brokers are alive)
- Controller election (one broker = active controller for partition assignment)
- Topic metadata storage (partition count, replication factor, config)
- Consumer group offset storage (moved to Kafka brokers in later versions)
- ISR tracking and notification

**KRaft (Kafka Raft) — ZooKeeper removal:**
- Kafka 3.x+ uses internal Raft-based quorum controller
- Metadata stored as an event log within Kafka itself
- Simplifies deployment (no separate ZooKeeper cluster)
- Faster controller failover (no ZooKeeper session timeout)

### Consumer Rebalancing Deep Dive

**Rebalance protocol:**

```
Consumer → hash(groupName) → Coordinator Broker
                                    ↓
                            Maintains member list
                            Detects joins/leaves/crashes via heartbeats
                                    ↓
                            Triggers rebalance
                                    ↓
                            All consumers rejoin
                                    ↓
                            Coordinator elects group leader
                                    ↓
                            Leader computes partition plan
                                    ↓
                            Coordinator broadcasts to all members
                                    ↓
                            Consumers resume from committed offsets
```

**Assignment strategies:**

| Strategy | Behavior |
|----------|----------|
| **RangeAssignor** | Divides partitions per topic into contiguous ranges per consumer |
| **RoundRobinAssignor** | Round-robin across all partitions of all subscribed topics |
| **StickyAssignor** | Minimizes partition movement during rebalance; preserves previous assignments |
| **CooperativeStickyAssignor** | Incremental rebalance; consumers don't stop all partitions during rebalance |

**Rebalance scenarios:**

1. **New consumer joins:** Coordinator notifies existing members on next heartbeat → full rejoin → leader reassigns
2. **Consumer leaves gracefully:** Sends leave-group request → immediate rebalance
3. **Consumer crashes:** Coordinator detects via heartbeat timeout (`session.timeout.ms`) → rebalance
4. **Partition count changes:** Producers notified on next metadata refresh; consumers trigger rebalance

**Static group membership:**
- Set `group.instance.id` on consumer
- Consumer restart within `session.timeout.ms` gets same partitions back without full rebalance
- Reduces unnecessary rebalances during rolling deploys

### Message Structure

| Field | Type | Description |
|-------|------|-------------|
| key | byte[] | Partition routing key; `hash(key) % numPartitions`. Optional |
| value | byte[] | Message payload (plain text or compressed binary) |
| topic | string | Topic name |
| partition | integer | Assigned partition ID |
| offset | long | Position within partition (monotonically increasing) |
| timestamp | long | When message was stored |
| headers | key-value[] | Optional metadata (tags, correlation IDs, tracing) |
| size | integer | Message size in bytes |
| crc | integer | Cyclic redundancy check for data integrity |

**Key vs partition:**
- Key is a business concept (user ID, order ID); partition is an infrastructure concept
- If key is null, producer round-robins across partitions
- Custom partitioner can override default `hash(key) % N` logic

### Producer Internals

```
Application Thread → RecordAccumulator (in-memory buffer, per-partition batches)
                              ↓
                     Sender Thread (I/O thread)
                              ↓
                     Batch sent to leader broker
                              ↓
                     ACK received (based on acks config)
```

**Key producer configs:**

| Config | Default | Effect |
|--------|---------|--------|
| `acks` | 1 | 0=fire-and-forget, 1=leader-only, all=full ISR |
| `retries` | MAX_INT | Number of retry attempts on transient failure |
| `batch.size` | 16384 | Max bytes per batch |
| `linger.ms` | 0 | Time to wait for more messages before sending batch |
| `buffer.memory` | 33554432 | Total memory for buffering; blocks producer if full |
| `max.in.flight.requests.per.connection` | 5 | Set to 1 for strict ordering with retries |
| `enable.idempotence` | false | Enables exactly-once producer semantics |
| `compression.type` | none | none, gzip, snappy, lz4, zstd |

### Segment Files & Log Compaction

**Segment structure:**
```
Partition-0/
├── 00000000000000000000.log      (data file)
├── 00000000000000000000.index    (offset → file position)
├── 00000000000000000000.timeindex (timestamp → offset)
├── 00000000000001000000.log      (next segment)
└── ...
```

**Log compaction** (alternative to time-based retention):
- Keeps latest value for each key; removes older duplicates
- Useful for changelogs, KV snapshots
- Compaction runs in background; active segment never compacted

### Exactly-Once Semantics (EOS) in Kafka

**Producer side — Idempotent producer:**
- Each producer gets a unique Producer ID (PID)
- Each message gets a sequence number per partition
- Broker deduplicates by (PID, partition, sequence)
- Enable: `enable.idempotence=true`

**End-to-end — Transactional API:**
```
producer.initTransactions()
producer.beginTransaction()
producer.send(record1)
producer.send(record2)
producer.sendOffsetsToTransaction(offsets, consumerGroupId)
producer.commitTransaction()  // atomic: all-or-nothing
```
- Atomic writes across multiple partitions and topics
- Consumer reads with `isolation.level=read_committed` (skips uncommitted/aborted)

---

## 2. Event Sourcing Implementation Details

### File-Based Architecture (High-Performance)

Optimization stack from Digital Wallet design:

| Layer | Implementation | Benefit |
|-------|---------------|---------|
| Command store | Local disk (mmap) | No network hop; sequential write |
| Event store | Local disk (mmap) | Append-only; OS page cache acceleration |
| State store | RocksDB (local LSM-tree) | Optimized for writes; recent data cached |
| Snapshot | Object storage (HDFS/S3) | Periodic full state dump for fast recovery |

**mmap optimization:**
- Maps disk file to memory as array
- OS caches file sections in memory automatically
- For append-only operations, virtually guaranteed in-memory performance
- Eliminates explicit read-from-disk after write

### Distributed Event Sourcing with Consensus

When local files create single-point-of-failure risk:

**Raft-based replication of event log:**
- Event list replicated across N nodes (typically 3 or 5)
- Raft guarantees: no data loss + same order across nodes
- Tolerates (N-1)/2 node failures
- Only the **event list** needs strong replication:
  - State can be reconstructed from events
  - Snapshots can be regenerated from events
  - Commands are non-deterministic (may produce different events) — not sufficient for replay

**Node roles:**

| Role | Count | Function |
|------|-------|----------|
| Leader | 1 | Receives commands, replicates events |
| Follower | N-1 | Receives replicated events, serves as hot standby |
| Candidate | transient | During leader election |

### Event Store Schema Options

**Append-only table (relational):**

| Column | Type | Description |
|--------|------|-------------|
| event_id | UUID | Unique event identifier |
| aggregate_id | UUID | Entity this event belongs to |
| aggregate_type | string | Type of entity (e.g., "Wallet", "Order") |
| event_type | string | e.g., "MoneyTransferred", "OrderPlaced" |
| payload | JSONB | Event data |
| metadata | JSONB | Correlation ID, causation ID, user ID |
| version | integer | Optimistic concurrency (per aggregate) |
| created_at | timestamp | Event timestamp |

**Optimistic concurrency:**
- Each event has a version number per aggregate
- On write: `INSERT ... WHERE version = expected_version`
- Conflict → retry command from latest state

### Snapshot Strategies

| Strategy | Trigger | Trade-off |
|----------|---------|-----------|
| **Count-based** | Every N events | Predictable; may be too frequent or too rare |
| **Time-based** | Every T minutes/hours | Aligns with business cycles (daily for finance) |
| **Size-based** | When event log exceeds S bytes | Controls storage growth |
| **On-demand** | Before known heavy replay | Manual but targeted |

**Snapshot structure:**
```
{
  "aggregate_id": "wallet-123",
  "version": 15000,
  "state": { "balance": 5432.10, "currency": "USD" },
  "created_at": "2025-01-01T00:00:00Z"
}
```

Recovery: load snapshot → replay events from `version + 1` onwards.

---

## 3. Message Filtering and Delayed/Scheduled Messages

### Message Filtering

**Tag-based filtering (broker-side):**
- Attach tags to message metadata (not payload)
- Consumer subscribes with tag filter
- Broker filters before delivering — reduces network traffic
- No deserialization needed (tags in metadata)

**Why not filter on payload:**
- Deserialization at broker degrades performance
- Sensitive data in payload shouldn't be readable by broker
- Keep filtering criteria in headers/tags

### Delayed Messages

```
Producer → Temporary storage (delay topic) → Timer → Target topic → Consumer
```

**Implementation approaches:**

| Approach | How it works | Precision |
|----------|-------------|-----------|
| **Predefined delay levels** | Separate internal topic per delay level (1s, 5s, 10s, 30s, 1m, ..., 2h). Used by RocketMQ | Fixed levels only |
| **Hierarchical timing wheel** | Efficient timer data structure; O(1) start/stop | Arbitrary precision |
| **Database polling** | Store scheduled messages in DB, poll for due messages | Depends on poll interval |

**Use case:** Order timeout — send delayed message at order creation; 30 minutes later consumer checks if payment completed, closes order if not.

### Scheduled Messages

Same architecture as delayed messages but with absolute timestamp instead of relative delay.

---

## 4. Exactly-Once with Idempotency Keys

### The Problem

At-least-once delivery means consumers may process the same message multiple times. Sources of duplication:
- Producer retry after timeout (message was actually delivered)
- Consumer crash after processing but before offset commit
- Rebalance during processing

### Idempotency Key Pattern

```
Message: { idempotency_key: "order-123-payment-v1", payload: {...} }

Consumer logic:
1. Check if idempotency_key exists in processed_keys store
2. If exists → skip (already processed)
3. If not → process message → store key → commit offset
```

**Storage for idempotency keys:**

| Store | Pros | Cons |
|-------|------|------|
| Database (unique constraint) | ACID, reliable | Additional DB write per message |
| Redis (SET NX with TTL) | Fast; auto-expiry | Not durable across restarts without persistence |
| Kafka compacted topic | Scales with Kafka | More complex to query |

**Key design considerations:**
- TTL on idempotency keys (don't store forever)
- Key must be deterministic from message content (not random UUID per delivery)
- Combine with database transactions: process + store key in same transaction

### Transactional Outbox Pattern

For services that need to update DB and publish event atomically:

```
1. BEGIN TRANSACTION
2. UPDATE orders SET status = 'paid' WHERE id = 123
3. INSERT INTO outbox (event_type, payload) VALUES ('OrderPaid', {...})
4. COMMIT

Background process:
5. Poll outbox table for unpublished events
6. Publish to Kafka
7. Mark as published (or delete)
```

- Guarantees DB state and event are consistent
- Outbox relay can use CDC (Debezium) instead of polling

---

## 5. DLQ Patterns and Retry Strategies

### Dead Letter Queue (DLQ)

Messages that fail processing after max retries are moved to a DLQ rather than blocking the consumer.

```
Main Topic → Consumer → [success] → commit offset
                      → [failure] → retry topic (with backoff)
                                        → [still failing after N retries]
                                        → DLQ topic
```

### Retry Topology

```
main-topic
├── retry-topic-1  (delay: 1 min)
├── retry-topic-2  (delay: 5 min)
├── retry-topic-3  (delay: 30 min)
└── dlq-topic      (manual intervention)
```

Each retry topic has increasing delay. After exhausting all retry topics, message lands in DLQ.

### Retry Strategies

| Strategy | Pattern | When to use |
|----------|---------|------------|
| **Immediate retry** | Retry N times with no delay | Transient network blips |
| **Linear backoff** | Wait T, 2T, 3T, ... | Moderate recovery time expected |
| **Exponential backoff** | Wait T, 2T, 4T, 8T, ... | Downstream service recovery |
| **Exponential + jitter** | Exponential delay ± random offset | Prevent thundering herd |

### DLQ Monitoring & Resolution

| Step | Action |
|------|--------|
| 1 | Alert on DLQ message count (monitor lag) |
| 2 | Inspect failed messages (log error reason, message content) |
| 3 | Fix root cause (bug, schema change, downstream outage) |
| 4 | Replay DLQ messages back to main topic |
| 5 | Verify processing success |

### Poison Message Handling

A poison message is one that will never be processable (malformed, schema incompatible):
- Detect via consistent failure across retries
- Move to DLQ with error metadata
- Don't block other messages in the partition
- Option: separate "poison" topic for automated analysis

---

## 6. Kafka Use Cases

| Use Case | How Kafka is used |
|----------|------------------|
| **Log processing** | Retain logs until expiration; consumers process at own pace |
| **Data streaming / recommendations** | Real-time feature pipelines feeding ML models |
| **System monitoring & alerting** | Metrics → Kafka → stream processors → alert system |
| **CDC (Change Data Capture)** | Debezium captures DB changes → Kafka → downstream consumers |
| **System migration** | Dual-write via Kafka during migration period |
| **Event sourcing** | Kafka as durable event store for microservices |
| **Stream processing** | Kafka Streams / Flink / Spark consume and transform in real-time |

---

## 7. Message Data Structure Contract

The message data structure is a contract between producers, brokers, and consumers. If any part of the system disagrees, messages must be mutated (copied), which kills performance.

**Design principles:**
- Message format should remain unchanged from producer → broker → consumer
- Minimize copying (zero-copy path)
- Use schema registry (Avro, Protobuf) for evolution without breaking contract
- Schema evolution rules: additive changes only (new optional fields), never remove/rename required fields

---

## 8. State Storage for Consumer Offsets

**Access pattern:**
- Frequent reads and writes (every commit)
- Small data volume
- Random access (per consumer group, per partition)
- Strong consistency required

**Evolution:**
1. ZooKeeper (original) — KV store, consistent
2. Kafka internal topic `__consumer_offsets` (current) — compacted topic, each key = (group, topic, partition), value = offset
3. Benefit: removes ZooKeeper dependency for hot path

---

## 9. Comparing Message Queue Technologies

| Feature | Kafka | RabbitMQ | Pulsar | SQS |
|---------|-------|----------|--------|-----|
| Model | Log-based streaming | Traditional MQ | Log-based (tiered storage) | Managed queue |
| Ordering | Per-partition | No guarantee (per-queue with single consumer) | Per-partition | FIFO queues available |
| Retention | Configurable (days-forever) | Until consumed | Tiered (hot/cold) | 14 days max |
| Replay | Yes (offset seek) | No | Yes | No |
| Throughput | Very high | Moderate | Very high | Moderate |
| Latency | Low-medium | Very low | Low | Medium |
| Protocol | Custom binary | AMQP, MQTT, STOMP | Custom binary | HTTP/SQS API |
| Managed options | Confluent, MSK | CloudAMQP | StreamNative | AWS native |
| Best for | Event streaming, CDC, logs | Task routing, RPC, low-latency | Multi-tenancy, geo-replication | Serverless, simple queuing |
