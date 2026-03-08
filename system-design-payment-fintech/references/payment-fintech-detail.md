# Payment & Fintech — Detailed Reference

## 1. Payment Flow State Machine

### Payment Order Status Transitions

```
NOT_STARTED ──→ EXECUTING ──→ SUCCESS ──→ wallet_updated=true ──→ ledger_updated=true
                    │                         │
                    └──→ FAILED               └──→ is_payment_done=true
                                                    (when ALL orders under same checkout_id succeed)
```

**Detailed state flow:**

1. `payment_order_status` = `NOT_STARTED` (initial)
2. Payment service sends order to executor → status = `EXECUTING`
3. Executor calls PSP:
   - PSP returns success → status = `SUCCESS`
   - PSP returns failure → status = `FAILED`
4. On `SUCCESS`: payment service calls wallet service → `wallet_updated = TRUE`
5. Wallet done → payment service calls ledger service → `ledger_updated = TRUE`
6. All payment orders for same `checkout_id` succeed → `is_payment_done = TRUE`

**Monitoring:** Scheduled job runs at fixed intervals, checks in-flight payment orders. Alerts when an order hasn't completed within threshold.

### Hosted Payment Page — Full 9-Step Flow

```
┌────────┐    ┌──────────────┐    ┌─────┐
│ Client │    │Payment Service│    │ PSP │
└───┬────┘    └──────┬───────┘    └──┬──┘
    │  1. checkout    │               │
    │────────────────►│               │
    │                 │ 2. register   │
    │                 │   payment     │
    │                 │──────────────►│
    │                 │  3. token     │
    │                 │◄──────────────│
    │                 │ 4. persist    │
    │                 │   token       │
    │  5. render PSP  │               │
    │     hosted page │               │
    │◄────────────────│               │
    │ 6. fill card details            │
    │────────────────────────────────►│
    │              7. payment status  │
    │◄────────────────────────────────│
    │ 8. redirect to                  │
    │    success URL                  │
    │                 │ 9. webhook    │
    │                 │   (async)     │
    │                 │◄──────────────│
    │                 │ update status │
```

**Key details:**
- Step 2: Registration includes amount, currency, expiration, redirect URL, and UUID/nonce (= payment_order_id)
- Step 3: Token is a UUID on PSP side, uniquely maps to the nonce, used for status lookups
- Step 5: PSP JavaScript uses token to retrieve payment details; needs token + redirect URL
- Step 8: Redirect URL format: `https://yourcompany.com/?tokenID=JIOUIQ123NSF&payResult=X324FSa`
- Step 9: Webhook URL registered with PSP during initial setup, receives payment events asynchronously

### Payment Processing Delays

Some payments take hours/days to complete:
- PSP flags high-risk → human review required
- 3D Secure Authentication → extra card holder verification

**Handling:**
1. PSP returns `pending` status → client displays to user + provides status-check page
2. PSP tracks pending payment → notifies payment service via webhook on status change
3. Some PSPs require payment service to poll for status updates instead

**Consistency requirement:** All 3 parties (client, payment service, PSP) must be eventually consistent. All communications must be idempotent + reconciliation ensures consistency.

## 2. Distributed Transaction Implementation Details

### 2PC (Two-Phase Commit) — Coordinator Flow

```
Phase 1: PREPARE
┌────────────┐     ┌──────┐     ┌──────┐
│ Coordinator│     │ DB A │     │ DB C │
│  (wallet   │     │(acct │     │(acct │
│  service)  │     │  A)  │     │  C)  │
└─────┬──────┘     └──┬───┘     └──┬───┘
      │  read/write   │            │
      │──────────────►│  (locked)  │
      │  read/write   │            │
      │───────────────────────────►│  (locked)
      │  prepare?     │            │
      │──────────────►│            │
      │  prepare?     │            │
      │───────────────────────────►│
      │  yes          │            │
      │◄──────────────│            │
      │  yes          │            │
      │◄───────────────────────────│

Phase 2: COMMIT (or ABORT if any "no")
      │  commit       │            │
      │──────────────►│  (unlock)  │
      │  commit       │            │
      │───────────────────────────►│  (unlock)
```

**Problems with 2PC:**
- Locks held during entire prepare → commit cycle (performance bottleneck)
- Coordinator is SPOF — if it crashes between prepare and commit, databases remain locked indefinitely
- Uses X/Open XA standard for heterogeneous database coordination
- Not performant for high-throughput systems

### TC/C (Try-Confirm/Cancel) — Detailed Flow

**Transfer $1 from Account A to Account C:**

```
Initial state: A=$1, C=$0

PHASE 1 — TRY:
┌────────────┐     ┌──────┐     ┌──────┐
│ Coordinator│     │ DB A │     │ DB C │
└─────┬──────┘     └──┬───┘     └──┬───┘
      │  A: -$1       │            │
      │──────────────►│            │
      │  (txn commits,│            │
      │   lock freed) │            │
      │  C: NOP       │            │
      │───────────────────────────►│
      │               │            │
State: A=$0, C=$0  (UNBALANCED — $1 "missing")

PHASE 2a — CONFIRM (if both Try succeed):
      │  A: NOP       │            │
      │──────────────►│            │
      │  C: +$1       │            │
      │───────────────────────────►│
      │  (txn commits)│            │
State: A=$0, C=$1  (BALANCED)

PHASE 2b — CANCEL (if any Try fails):
      │  A: +$1       │            │
      │──────────────►│  (reverse) │
      │  C: NOP       │            │
      │───────────────────────────►│
State: A=$1, C=$0  (RESTORED)
```

**Critical design decisions:**
- Only Choice 1 is valid for Try phase: deduct from source, NOP on destination
- Choice 2 (NOP on source, credit destination) fails because someone could spend the credited $1 before cancel
- Choice 3 (both deduct and credit) creates complex partial-failure scenarios

### Phase Status Table

Stored in the same database as the deducting account (account A). Contains:

| Field | Description |
|-------|-------------|
| distributed_txn_id | Unique ID for the distributed transaction |
| distributed_txn_content | Details of the transaction |
| try_phase_status_per_db | "not sent yet" / "has been sent" / "response received" per database |
| second_phase_name | "Confirm" or "Cancel" (derived from Try results) |
| second_phase_status | Status of confirm/cancel execution |
| out_of_order_flag | Handles network reordering (Cancel arrives before Try) |

**Recovery:** If coordinator crashes, it restarts and reads phase status table to determine where to resume.

### Out-of-Order Execution Handling

Network issues can cause Cancel to arrive before Try at a database node:

1. Cancel arrives first → node sets out-of-order flag in DB (nothing to cancel yet)
2. Try arrives later → checks for out-of-order flag → returns failure immediately

This ensures TC/C correctness even under network reordering.

### 2PC vs TC/C — Key Differences

| Aspect | 2PC | TC/C |
|--------|-----|------|
| Phase 1 lock state | Local transactions NOT done (still locked) | Local transactions DONE (committed, unlocked) |
| Phase 2 success | Complete unfinished transactions | Execute new transactions if needed |
| Phase 2 failure | Cancel uncommitted transactions | Reverse already-committed transactions |
| Database requirement | XA support required | Any transactional DB works |
| Implementation layer | Database | Application business logic |
| Intermediate inconsistency | Hidden by DB locks | Visible to application (unbalanced state) |

### Saga — Compensation Flow

```
Normal flow (left to right):
  A: -$1 ──→ C: +$1 ──→ SUCCESS

Error at step 2:
  A: -$1 ──→ C: +$1 FAILS
                │
                ▼
  Compensate:   C: NOP ──→ A: +$1 (reverse) ──→ ERROR returned

Error at step 1:
  A: -$1 FAILS ──→ ERROR returned (no compensation needed)
```

**Coordination modes:**

| Mode | Description | Trade-off |
|------|-------------|-----------|
| Choreography | Services subscribe to each other's events, fully decentralized | Each service maintains internal state machine; hard to manage with many services |
| Orchestration | Single coordinator instructs services in order | Handles complexity well; preferred for wallet systems |

**Saga requires 2n operations:** n for normal case + n for compensating transactions during rollback.

### TC/C vs Saga Decision Matrix

| Factor | Choose TC/C | Choose Saga |
|--------|------------|-------------|
| Latency requirement | Strict — needs parallel execution | Relaxed — linear is acceptable |
| Number of services | Many — parallel TC/C reduces total time | Few — linear overhead is minimal |
| Architecture trend | Custom fintech | Microservice ecosystem |
| Coordination | Always centralized | Choreography or orchestration |

## 3. Stock Exchange Trading Flow — End-to-End

### Order Lifecycle

```
                            ┌─────────────────────────────────────────┐
                            │         EXCHANGE (single server)         │
Broker ──FIX──► Client  ──► Order    ──► Sequencer ──► Matching  ──► Sequencer
                Gateway     Manager      (inbound      Engine        (outbound
                │           │            seq IDs)      │             seq IDs)
                │           ├─ Risk                    │
                │           │  Check                   ├──► MDP ──► Data Service
                │           └─ Wallet                  │
                │              Check                   └──► Reporter ──► DB
                │                                              │
                └◄──────── Executions (fills) ◄────────────────┘
```

### Critical Trading Path

The path with strict latency requirements:

```
Client Gateway → Order Manager → Sequencer (inbound) → Matching Engine → Sequencer (outbound) → Order Manager → Client Gateway
```

**What happens at each step:**

1. **Client Gateway:** FIX message received → validated → rate-limited → authenticated → normalized → forwarded
2. **Order Manager (inbound):**
   - Runs risk checks (e.g., max 1M shares/day per user)
   - Verifies wallet has sufficient funds (funds withheld for pending orders)
   - Strips unnecessary attributes → sends minimal message to sequencer
3. **Sequencer (inbound):** Stamps sequential ID → any gap in sequence = detectable missing order
4. **Matching Engine:** Checks order book → attempts match → emits 0 or 2 executions per match
5. **Sequencer (outbound):** Stamps sequential ID on executions
6. **Order Manager (outbound):** Updates order state, routes executions back to client

### Market Data Flow

```
Matching Engine → executions → Market Data Publisher → Data Service → Brokers → Clients
```

MDP receives execution stream and reconstructs:
- Order book (L1/L2/L3 data)
- Candlestick charts (multiple time intervals)

### Reporting Flow (not on critical path)

Reporter merges attributes from both incoming orders and outgoing executions:
- Incoming order: full order details
- Outgoing execution: order ID, price, quantity, execution status
- Combined: complete records for compliance, tax reporting, settlement, reconciliation

## 4. Matching Engine Algorithms

### FIFO Matching (Price-Time Priority)

```java
Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

**Order handling dispatch:**

```java
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    // Verify sequence continuity
    if (orderEvent.getSequenceId() != nextSequence)
        return Error(OUT_OF_ORDER, nextSequence);
    // Validate order fields
    if (!validateOrder(symbol, price, quantity))
        return ERROR(INVALID_ORDER, orderEvent);

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:    return handleNew(orderBook, order);
        case CANCEL: return handleCancel(orderBook, order);
}
```

**New order:** Route to opposite book (buy order matches against sell book).

**Cancel order:** Look up in `orderMap` → if found, remove from price level → set status CANCELED. If already matched → return `CANNOT_CANCEL_ALREADY_MATCHED`.

### Order Book Operations — O(1) Complexity

| Operation | How | Complexity |
|-----------|-----|-----------|
| Place new order | Append to tail of PriceLevel (doubly-linked list) | O(1) |
| Match order | Remove from head of PriceLevel | O(1) |
| Cancel order | Lookup via `Map<OrderID, Order>` → remove from doubly-linked list | O(1) |
| Query best bid/ask | Direct reference stored in OrderBook | O(1) |

### FIFO + LMM (Lead Market Maker)

LMM firms negotiate a predefined allocation ratio with the exchange. At each price level, the LMM gets their allocation filled before FIFO queue is processed. Incentivizes market makers to provide liquidity.

## 5. Candlestick Chart Data Generation

### Data Structure

```java
class Candlestick {
    long openPrice;    // first trade price in interval
    long closePrice;   // last trade price in interval
    long highPrice;    // highest trade price in interval
    long lowPrice;     // lowest trade price in interval
    long volume;       // total shares traded in interval
    long timestamp;    // start of the interval
    int interval;      // interval duration in seconds
}

class CandlestickChart {
    LinkedList<Candlestick> sticks;
}
```

### Generation Process

1. MDP receives execution stream from matching engine
2. For each execution, update current candlestick:
   - If first trade in interval → set open, high, low, close to trade price
   - If subsequent trade → update close to trade price, update high/low if applicable, add to volume
3. When interval elapses → finalize current candlestick, create new one

### Common intervals

1-minute, 5-minute, 1-hour, 1-day, 1-week, 1-month. Each symbol maintains separate candlestick charts per interval.

### Memory Optimization

- **Ring buffers:** Pre-allocated circular buffer holds N most recent candlesticks per interval. No object creation/deallocation overhead. Lock-free, cache-line padded for CPU efficiency.
- **Disk persistence:** Older candlesticks flushed to disk. In-memory columnar DB (e.g., KDB) for real-time analytics. Historical DB for post-market storage.

## 6. FIX Protocol Message Format

### Structure

FIX messages are sequences of tag=value pairs separated by delimiter (`|` or SOH character).

```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS |
52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 |
20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 |
38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING |
59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 |
10=128 |
```

### Key Tags

| Tag | Name | Description | Example values |
|-----|------|-------------|---------------|
| 8 | BeginString | FIX version | FIX.4.2 |
| 9 | BodyLength | Message body length | 176 |
| 35 | MsgType | Message type | 8=ExecutionReport, D=NewOrderSingle |
| 49 | SenderCompID | Sender identifier | PHLX |
| 56 | TargetCompID | Target identifier | PERS |
| 52 | SendingTime | Timestamp | 20071123-05:30:00.000 |
| 55 | Symbol | Trading symbol | MSFT |
| 54 | Side | Buy/Sell | 1=Buy, 2=Sell |
| 38 | OrderQty | Order quantity | 15 |
| 40 | OrdType | Order type | 1=Market, 2=Limit |
| 44 | Price | Limit price | 15 |
| 39 | OrdStatus | Order status | 0=New, 1=PartialFill, 2=Filled, 4=Canceled |
| 150 | ExecType | Execution type | 0=New, F=Trade |
| 10 | CheckSum | Message checksum | 128 |

### Message Types Used in Exchange

| MsgType (35) | Name | Direction |
|-------------|------|-----------|
| D | New Order Single | Client → Exchange |
| F | Order Cancel Request | Client → Exchange |
| 8 | Execution Report | Exchange → Client |
| 9 | Order Cancel Reject | Exchange → Client |

### Internal Optimization

For internal exchange communication, FIX is transformed to **FIX over Simple Binary Encoding (SBE)** for fast and compact encoding. SBE uses fixed-size fields and no delimiters, enabling zero-copy parsing.

## 7. Market Data Publisher Architecture

### Full Architecture

```
Matching Engine
      │
      ▼ (execution stream)
┌─────────────────────────────────┐
│   Market Data Publisher (MDP)   │
│                                 │
│  ┌────────────────────────────┐ │
│  │   Order Book Rebuilder     │ │
│  │   (L1, L2, L3 per symbol) │ │
│  └────────────┬───────────────┘ │
│               │                 │
│  ┌────────────▼───────────────┐ │
│  │   Candlestick Generator    │ │
│  │   (ring buffers per        │ │
│  │    symbol × interval)      │ │
│  └────────────┬───────────────┘ │
│               │                 │
│  ┌────────────▼───────────────┐ │
│  │   Subscriber Manager       │ │
│  │   (multicast groups)       │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
      │
      ▼ (market data)
┌─────────────┐
│ Data Service │──► Brokers ──► Clients
└─────────────┘
```

### Ring Buffer Implementation

```
                write
                  │
    ┌─────────────▼──────────────┐
    │ [ ][ ][ ][ ][ ][ ][ ][ ] │
    │  ▲                     ▲   │
    │  │                     │   │
    │  read (consumer 1)     │   │
    │       read (consumer 2)    │
    └────────────────────────────┘
         (head connects to tail)
```

**Properties:**
- Fixed-size, pre-allocated memory
- No object creation/deallocation (critical for low-latency)
- Lock-free with memory barriers
- Cache-line padded sequence numbers (prevent false sharing)
- Producer writes sequentially; consumers read at their own pace

### Distribution Fairness

**Problem:** If MDP iterates subscriber list sequentially, first subscriber always gets data first → unfair advantage in trading.

**Solutions:**
1. **Multicast (reliable UDP):** Broadcast to multicast group → all subscribers receive simultaneously
2. **Random subscriber ordering:** Assign random position when subscriber connects
3. **Separate multicast groups per data tier:** Retail (L1/L2) vs institutional (L3)

### Tiered Market Data Access

| Tier | Data | Typical users |
|------|------|---------------|
| Free | Delayed L1 (15-20 min delay) | Casual investors |
| Basic | Real-time L1 | Retail traders |
| Standard | Real-time L2 (5 levels) | Active retail traders |
| Premium | Real-time L2 (10 levels) | Day traders |
| Professional | Real-time L3 | Institutional, hedge funds |

### Event Sourcing Integration

MDP uses event sourcing design:
- Receives `OrderFilledEvent` from event store
- Rebuilds order book by applying events to order book state machine
- Order manager is embedded as a reusable library within MDP (avoids centralized queries)
- States guaranteed identical across components because same event stream is replayed deterministically

### Hot-Warm Failover for MDP

```
Hot MDP (primary)
  │ processes events, publishes market data
  │
  ├──► Event Store (mmap) ──► Warm MDP (standby)
  │                            processes events, does NOT publish
  │
  └── heartbeat ──► if missing ──► warm takes over publishing
```

Recovery: warm instance loads latest snapshot + replays events from event store to reach current state.
