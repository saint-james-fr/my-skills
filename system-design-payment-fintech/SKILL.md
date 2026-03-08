---
name: system-design-payment-fintech
description: Payment and fintech system design — payment processing, digital wallets, stock exchanges, double-entry ledger, distributed transactions (2PC, Saga, TC/C), PSP integration, reconciliation, and matching engines. Use when designing payment systems, implementing digital wallets, building trading platforms, or handling distributed financial transactions.
---

# Payment & Fintech System Design

## 1. Payment System Design

### Pay-in Flow

```
Client → Payment Service → Payment Executor → PSP → Card Scheme → Bank
                │                                          │
                ├──→ Wallet (update seller balance) ◄──────┘
                └──→ Ledger (record transaction)
```

| Component | Role |
|-----------|------|
| Payment service | Accepts payment events, orchestrates flow, runs AML/CFT risk check via third-party |
| Payment executor | Executes a single payment order through PSP. One event may contain multiple orders |
| PSP | Moves money from buyer to e-commerce bank account (Stripe, Braintree, Square) |
| Card scheme | Processes credit card operations (Visa, MasterCard) |
| Ledger | Financial record — every transaction logged as debit + credit |
| Wallet | Tracks merchant account balance |

**Pay-in steps:**
1. User clicks "place order" → payment event sent to payment service
2. Payment service stores event in DB
3. For each payment order, calls payment executor
4. Executor stores order in DB, calls PSP
5. On success → update wallet (seller balance) → update ledger

### Pay-out Flow

Money flows from e-commerce bank account → seller bank account via third-party pay-out provider (e.g. Tipalti). Triggered when products are delivered and money is released.

### Payment API

**POST /v1/payments** — execute a payment event

| Field | Type | Notes |
|-------|------|-------|
| buyer_info | json | Buyer details |
| checkout_id | string | Globally unique checkout ID |
| credit_card_info | json | Encrypted card info or payment token (PSP-specific) |
| payment_orders | list | Array of payment orders |

Each payment order:

| Field | Type | Notes |
|-------|------|-------|
| seller_account | string | Recipient |
| amount | **string** | Never use double — precision loss risk; parse only for display/calculation |
| currency | string | ISO 4217 |
| payment_order_id | string | Globally unique — used as PSP idempotency key |

**GET /v1/payments/{:id}** — query payment order status

### Data Model

**payment_event table:**

| Column | Type |
|--------|------|
| checkout_id | string PK |
| buyer_info | string |
| seller_info | string |
| credit_card_info | depends on provider |
| is_payment_done | boolean |

**payment_order table:**

| Column | Type |
|--------|------|
| payment_order_id | string PK |
| buyer_account | string |
| amount | string |
| currency | string |
| checkout_id | string FK |
| payment_order_status | enum (NOT_STARTED, EXECUTING, SUCCESS, FAILED) |
| ledger_updated | boolean |
| wallet_updated | boolean |

Storage choice: traditional relational DB with ACID — proven stability, rich tooling, mature DBA market.

## 2. PSP Integration

Two integration models:

| Model | Who stores card data | PCI burden | Typical user |
|-------|---------------------|------------|-------------|
| API integration | Merchant | Full PCI DSS compliance | Large companies with dedicated security teams |
| Hosted payment page | PSP | PSP handles PCI | Most companies (recommended) |

### Hosted Payment Page Flow

1. Client clicks "checkout" → payment service receives order
2. Payment service registers order with PSP (includes amount, currency, nonce/UUID, redirect URL)
3. PSP returns token (maps 1:1 to payment order via nonce)
4. Payment service persists token
5. Client renders PSP-hosted payment page (iframe/SDK) using token
6. User enters card details directly on PSP page → PSP processes payment
7. PSP returns status → browser redirects to merchant redirect URL
8. PSP asynchronously calls payment service webhook with final status

Token = idempotency key on PSP side. Same token → same payment, prevents double charge.

## 3. Double-Entry Ledger

Every payment records two entries that sum to zero:

| Account | Debit | Credit |
|---------|-------|--------|
| Buyer | $1 | |
| Seller | | $1 |

**Principles:**
- Sum of all transaction entries = 0
- One cent lost → someone else gains a cent
- Provides end-to-end traceability
- Entries are append-only and immutable
- Foundation for reconciliation and auditing

## 4. Reconciliation

Last line of defense for payment correctness.

**Process:**
1. PSP/bank sends daily settlement file (balance + all transactions for the day)
2. Reconciliation system parses file and compares with internal ledger
3. Also verifies internal consistency (ledger ↔ wallet states)

**Handling discrepancies:**

| Category | Description | Resolution |
|----------|-------------|------------|
| Classifiable + automatable | Known cause, cost-effective to automate | Automated fix |
| Classifiable + manual | Known cause, automation too expensive | Finance team queue |
| Unclassifiable | Unknown cause | Special investigation queue |

## 5. Payment Reliability

### Idempotency

Prevents double-charging. `exactly-once = at-least-once (retry) + at-most-once (idempotency)`.

**Implementation:**
- Client sends idempotency key in HTTP header: `Idempotency-Key: <uuid>`
- For e-commerce: idempotency key = shopping cart ID at checkout
- DB primary key serves as idempotency key — duplicate insert fails with unique constraint violation
- Concurrent same-key requests → return `429 Too Many Requests`

**Scenario 1 — User double-clicks "Pay":** Second request recognized by idempotency key, returns existing status.

**Scenario 2 — PSP succeeds but response lost:** Token (mapped from nonce) is reused → PSP recognizes duplicate via token-as-idempotency-key.

### Retry Strategies

| Strategy | Description |
|----------|-------------|
| Immediate | Retry instantly |
| Fixed interval | Wait fixed time between retries |
| Incremental | Increase wait time linearly |
| Exponential backoff | Double wait time each retry (1s, 2s, 4s, 8s...) |
| Cancel | Abort if failure is permanent |

Best practice: exponential backoff + `Retry-After` response header.

### Retry Queue & Dead Letter Queue

```
Payment fails → retryable?
  ├─ Yes → retry queue → retry → still fails?
  │                                ├─ Under threshold → back to retry queue
  │                                └─ Over threshold → dead letter queue (investigate)
  └─ No → store error in DB
```

### Sync vs Async Communication

| Aspect | Synchronous (HTTP) | Asynchronous (message queue) |
|--------|--------------------|------------------------------|
| Simplicity | Simpler design | More complex |
| Coupling | Tight | Loose |
| Performance | Chain-limited (slowest service) | Each service independent |
| Failure isolation | Poor — one failure breaks chain | Good — queue buffers failures |
| Scalability | Hard to scale | Easy — queue absorbs spikes |
| Best for | Small-scale | Large-scale payment systems |

**Async patterns:** Single receiver (shared queue, message removed after processing) vs Multiple receivers (Kafka — message stays, consumed by many services: payment, analytics, billing).

### Payment State Tracking

Status persisted in append-only table. Scheduled job monitors in-flight orders and alerts if not completed within threshold.

## 6. Digital Wallet Design

### API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| /v1/wallet/balance_transfer | POST | Transfer between wallets |

Request: `from_account`, `to_account`, `amount` (string), `currency` (ISO 4217)

### Balance Management Approaches

| Approach | Pros | Cons |
|----------|------|------|
| In-memory (Redis) | Fast | Not durable, no transactional guarantee |
| Relational DB + distributed transactions | ACID, durable | Performance limited per node |
| Event sourcing | Reproducible, auditable, immutable history | More complex architecture |

### In-Memory Sharding

For 1M TPS target: partition accounts across Redis nodes using hash-based sharding.

```
partition = accountID.hashCode() % partitionNumber
```

Zookeeper stores partition count + Redis node addresses. Wallet service is stateless, scales horizontally.

**Limitation:** No atomic cross-node transactions → replace Redis with relational DB + distributed transaction protocol.

### Event Sourcing for Wallets

```
Command → validate → Event → apply → State update
  (FIFO)              (FIFO)          (balance DB)
```

- Commands: balance transfer requests (may be invalid)
- Events: validated facts, past tense ("transferred $1"), deterministic
- State: account balances (key-value: account → balance)
- State Machine: validates commands, generates events, applies events — must be deterministic

**Reproducibility:** replay all events from beginning → reconstruct any historical state. Immutable event list + deterministic state machine = guaranteed identical replays.

**CQRS:** Publish events (not state). External read-only state machines build custom views (balance queries, audit trails). Eventually consistent.

**High-performance optimizations:**
- File-based command/event lists (local disk, sequential writes)
- mmap for memory-mapped I/O (avoid network transit to remote Kafka/DB)
- RocksDB for local state (LSM-optimized writes)
- Snapshots: periodic state checkpoints → avoid replaying from beginning

## 7. Distributed Transactions

| Aspect | 2PC | TC/C (Try-Confirm/Cancel) | Saga |
|--------|-----|---------------------------|------|
| Mechanism | DB-level (XA protocol) | Application-level compensation | Application-level sequential operations |
| First phase | Local txns NOT committed (locked) | Local txns committed | Operations executed one by one |
| Second phase success | Commit all | Execute new txns if needed | — (all done) |
| Second phase fail | Abort all (rollback) | Reverse committed txns ("undo") | Compensating txns in reverse order |
| Consistency | Strong (locks held) | Eventually consistent | Eventually consistent |
| Availability | Low (locks block) | Higher | Higher |
| Parallel execution | No | Yes (any order) | No (linear only) |
| Rollback | DB handles | Business logic "undo" | Compensating transactions |
| Coordinator SPOF | Yes | Phase status table for recovery | Orchestrator or choreography |
| Best use case | Small-scale, DB-supported XA | Latency-sensitive, many services | Microservices, few services |

**TC/C example** (transfer $1 from A → C):
- Try: deduct $1 from A (commit), NOP on C
- Confirm: NOP on A, add $1 to C (commit)
- Cancel: add $1 back to A (reverse), NOP on C

**Saga modes:** choreography (fully decentralized, services subscribe to events) vs orchestration (central coordinator, preferred for wallet systems).

**Choice heuristic:** Few services or no latency requirement → Saga. Latency-sensitive + many services → TC/C.

## 8. Stock Exchange Design

### Core Components

```
Broker → Client Gateway → Order Manager → Sequencer → Matching Engine
              │                │                            │
              │           Risk Manager                      ├─→ Market Data Publisher → Data Service
              │           Wallet Check                      └─→ Reporter → DB
              │                                             
              └──────────────── Executions (fills) ◄────────┘
```

| Component | Role |
|-----------|------|
| Client gateway | Input validation, rate limiting, auth, normalization |
| Order manager | Risk checks, wallet verification, state management via event sourcing |
| Sequencer | Stamps sequence IDs on orders (inbound) and executions (outbound), acts as event store |
| Matching engine | Maintains order book per symbol, matches buy/sell, emits executions |
| Market data publisher | Builds order books + candlestick charts from execution stream |
| Reporter | Merges order + execution attributes for compliance, tax, settlement |

### Order Types

| Type | Description |
|------|-------------|
| Limit order | Buy/sell at a fixed price — may not match immediately, may partially fill |
| Market order | Execute at prevailing price immediately — guarantees execution, not price |

### Order Book

List of buy/sell orders per symbol, organized by price level.

**Data structure requirements:** O(1) lookup by price level, O(1) add/cancel/match (doubly-linked list per price level), O(1) cancel via `Map<OrderID, Order>`.

```
OrderBook
├── buyBook: Map<Price, PriceLevel>
├── sellBook: Map<Price, PriceLevel>
├── bestBid, bestOffer
└── orderMap: Map<OrderID, Order>

PriceLevel
├── totalVolume: long
└── orders: DoublyLinkedList<Order>
```

### FIX Protocol

Financial Information eXchange — vendor-neutral protocol for securities transactions (since 1991). Tag-value pairs separated by `|`:

```
8=FIX.4.2 | 35=8 | 49=PHLX | 56=PERS | 55=MSFT | 54=1 | 38=15 | 40=2 | 44=15
```

Key tags: 35=MsgType, 49=SenderCompID, 55=Symbol, 54=Side(1=Buy), 38=Quantity, 44=Price.

### Matching Algorithms

| Algorithm | Description |
|-----------|-------------|
| FIFO (price-time priority) | First order at a price level matched first |
| FIFO + LMM | Lead Market Maker gets predefined allocation ahead of FIFO queue |

### Market Data Levels

| Level | Content |
|-------|---------|
| L1 | Best bid/ask price + quantities |
| L2 | Multiple price levels beyond best bid/ask |
| L3 | All price levels + queued quantity per order at each level |

## 9. Stock Exchange Performance

### Single-Server Architecture

Modern low-latency exchanges put all critical-path components on **one server**:

```
gateway → order manager → sequencer → matching engine
              (all on same server, communicating via mmap)
```

**Why:** Eliminates network hops (500μs per round-trip). mmap over `/dev/shm` = sub-microsecond IPC with zero disk access.

### Application Loop Pattern

- Single-threaded, pinned to a fixed CPU core per component
- No context switching, no locks, no lock contention
- Guarantees low 99th percentile latency

### Determinism

| Type | How achieved |
|------|-------------|
| Functional | Sequencer + event sourcing → same input order = same output |
| Latency | Low p99 latency via CPU pinning, mmap, no GC pauses |

### Event Sourcing for Recovery

Immutable log of all events. Replay events → reconstruct exact state. Hot-warm matching engine: warm instance processes same events but doesn't emit; takes over instantly on primary failure.

### High Availability

- Raft consensus across 5 servers (tolerates 2 failures)
- Leader replicates events to followers via RPC
- Leader election on heartbeat timeout
- RTO: seconds. RPO: near zero (Raft consensus guarantees)

### Ring Buffers for Market Data

Pre-allocated circular buffers — no object creation/deallocation, lock-free, cache-line padded. Used by Market Data Publisher for candlestick chart storage.

### Network

| Technique | Purpose |
|-----------|---------|
| Multicast (reliable UDP) | Broadcast market data to all subscribers simultaneously (fairness) |
| Colocation | Broker servers in exchange data center — latency ∝ cable length |

### DDoS Mitigation

1. Isolate public from private services
2. Cache infrequently-updated data
3. Harden URLs (avoid query-string-based enumeration)
4. Safelist/blocklist at network gateway
5. Rate limiting

## 10. Payment Security

| Threat | Mitigation |
|--------|------------|
| Eavesdropping | HTTPS |
| Data tampering | Encryption + integrity monitoring |
| Man-in-the-middle | SSL with certificate pinning |
| Password storage | Salted hashing |
| Data loss | Multi-region DB replication + snapshots |
| DDoS | Rate limiting + firewall |
| Card theft | Tokenization — store tokens instead of real card numbers |
| PCI compliance | PCI DSS standard; use hosted payment page to avoid handling card data |
| Fraud | Address verification, CVV, user behavior analysis |

### Payments Ecosystem (Visa Model)

```
Cardholder ($100) → Merchant → Acquiring Bank → Card Network (Visa) → Issuing Bank
                     keeps $97.75   keeps $0.25     assessment fee      interchange $1.75
```

Issuing bank compensated because: pays merchant before cardholder pays bank, absorbs default risk, manages fraud detection/risk/settlement.

### QR Code Payment Flow

1. Cashier calculates total → sends order ID + amount to PSP
2. PSP generates QR code URL → displayed at checkout
3. Consumer scans QR with wallet app → confirms amount → clicks "pay"
4. Wallet app notifies PSP → PSP marks QR as paid
5. PSP notifies merchant of payment
