# DDD & Clean Architecture — Detail Reference

## Aggregate Lifecycle: Full TypeScript Example

### Defining the Aggregate

```typescript
type TicketId = string & { readonly brand: unique symbol }
const TicketId = (id: string) => id as TicketId

type AgentId = string & { readonly brand: unique symbol }
type CustomerId = string & { readonly brand: unique symbol }

type TicketPriority = 'low' | 'medium' | 'high' | 'urgent'

type TicketStatus = 'open' | 'escalated' | 'resolved' | 'closed'

type Message = {
  readonly from: AgentId | CustomerId
  readonly body: string
  readonly sentAt: Date
  readonly wasRead: boolean
}

type Ticket = {
  readonly id: TicketId
  version: number
  status: TicketStatus
  priority: TicketPriority
  customer: CustomerId
  assignedAgent: AgentId
  messages: Message[]
  escalation: { reason: string; escalatedAt: Date } | null
  closedAt: Date | null
  domainEvents: DomainEvent[]
}

type DomainEvent =
  | { type: 'TicketOpened'; ticketId: TicketId; customer: CustomerId; priority: TicketPriority; occurredAt: Date }
  | { type: 'TicketEscalated'; ticketId: TicketId; reason: string; occurredAt: Date }
  | { type: 'TicketResolved'; ticketId: TicketId; agentId: AgentId; occurredAt: Date }
  | { type: 'TicketClosed'; ticketId: TicketId; occurredAt: Date }
```

### Commands as Pure Functions

```typescript
const openTicket = (
  id: TicketId,
  customer: CustomerId,
  agent: AgentId,
  priority: TicketPriority,
  body: string,
): Ticket => {
  const now = new Date()
  return {
    id,
    version: 0,
    status: 'open',
    priority,
    customer,
    assignedAgent: agent,
    messages: [{ from: customer, body, sentAt: now, wasRead: false }],
    escalation: null,
    closedAt: null,
    domainEvents: [{ type: 'TicketOpened', ticketId: id, customer, priority, occurredAt: now }],
  }
}

const escalate = (ticket: Ticket, reason: string): Ticket => {
  if (ticket.status === 'closed') throw new Error('Cannot escalate a closed ticket')
  if (ticket.escalation) throw new Error('Already escalated')

  const now = new Date()
  return {
    ...ticket,
    status: 'escalated',
    escalation: { reason, escalatedAt: now },
    domainEvents: [
      ...ticket.domainEvents,
      { type: 'TicketEscalated', ticketId: ticket.id, reason, occurredAt: now },
    ],
  }
}

const addMessage = (ticket: Ticket, from: AgentId | CustomerId, body: string): Ticket => {
  if (ticket.status === 'closed') throw new Error('Cannot message on a closed ticket')

  return {
    ...ticket,
    messages: [...ticket.messages, { from, body, sentAt: new Date(), wasRead: false }],
  }
}

const resolve = (ticket: Ticket, agentId: AgentId): Ticket => {
  if (ticket.status === 'closed') throw new Error('Already closed')
  if (ticket.escalation && agentId === ticket.assignedAgent) {
    throw new Error('Escalated tickets can only be resolved by a manager')
  }

  const now = new Date()
  return {
    ...ticket,
    status: 'resolved',
    domainEvents: [
      ...ticket.domainEvents,
      { type: 'TicketResolved', ticketId: ticket.id, agentId, occurredAt: now },
    ],
  }
}
```

### Application Layer (Use Case Orchestration)

```typescript
type TicketRepository = {
  load: (id: TicketId) => Promise<Ticket>
  save: (ticket: Ticket) => Promise<void>
}

const escalateTicketUseCase = async (
  repo: TicketRepository,
  ticketId: TicketId,
  reason: string,
): Promise<{ success: boolean; error?: string }> => {
  try {
    const ticket = await repo.load(ticketId)
    const updated = escalate(ticket, reason)
    await repo.save(updated)
    return { success: true }
  } catch (err) {
    if (err instanceof ConcurrencyError) {
      return { success: false, error: 'Ticket was modified by another user. Retry.' }
    }
    throw err
  }
}
```

### Repository Implementation (Infrastructure Layer)

```typescript
import { Pool } from 'pg'

const createPostgresTicketRepo = (pool: Pool): TicketRepository => ({
  load: async (id) => {
    const { rows } = await pool.query('SELECT data, version FROM tickets WHERE id = $1', [id])
    if (rows.length === 0) throw new Error(`Ticket ${id} not found`)
    return { ...rows[0].data, version: rows[0].version, domainEvents: [] }
  },

  save: async (ticket) => {
    const result = await pool.query(
      `UPDATE tickets
       SET data = $1, version = version + 1
       WHERE id = $2 AND version = $3`,
      [JSON.stringify(ticket), ticket.id, ticket.version],
    )

    if (result.rowCount === 0) throw new ConcurrencyError(ticket.id)

    if (ticket.domainEvents.length > 0) {
      const values = ticket.domainEvents.map((e, i) =>
        `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`
      ).join(', ')

      const params = ticket.domainEvents.flatMap((e) => [
        ticket.id,
        e.type,
        JSON.stringify(e),
      ])

      await pool.query(
        `INSERT INTO outbox (aggregate_id, event_type, payload) VALUES ${values}`,
        params,
      )
    }
  },
})
```

## Outbox Pattern — Full Implementation

### Database Schema

```sql
CREATE TABLE outbox (
  id          BIGSERIAL PRIMARY KEY,
  aggregate_id TEXT NOT NULL,
  event_type  TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  published   BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published = FALSE;
```

### Message Relay (Polling Publisher)

```typescript
const startOutboxRelay = (pool: Pool, messageBus: MessageBus, intervalMs = 1000) => {
  const poll = async () => {
    const { rows } = await pool.query(
      'SELECT id, event_type, payload FROM outbox WHERE published = FALSE ORDER BY id LIMIT 100',
    )

    for (const row of rows) {
      await messageBus.publish(row.event_type, row.payload)
      await pool.query('UPDATE outbox SET published = TRUE WHERE id = $1', [row.id])
    }
  }

  return setInterval(poll, intervalMs)
}
```

**At-least-once delivery:** if the relay crashes after publishing but before marking as published, the same event is published again. Consumers must be idempotent (e.g., use event ID for deduplication).

## Anticorruption Layer (ACL)

Translates an external bounded context's model into your own. Protects your domain from upstream model changes.

```typescript
type ExternalPaymentEvent = {
  txn_id: string
  amt: number
  ccy: string
  status: 'APPROVED' | 'DECLINED' | 'PENDING'
  ts: number
}

type PaymentConfirmed = {
  type: 'PaymentConfirmed'
  paymentId: string
  amount: Money
  confirmedAt: Date
}

type PaymentRejected = {
  type: 'PaymentRejected'
  paymentId: string
  rejectedAt: Date
}

const translatePaymentEvent = (
  ext: ExternalPaymentEvent,
): PaymentConfirmed | PaymentRejected | null => {
  switch (ext.status) {
    case 'APPROVED':
      return {
        type: 'PaymentConfirmed',
        paymentId: ext.txn_id,
        amount: { amount: ext.amt / 100, currency: ext.ccy },
        confirmedAt: new Date(ext.ts * 1000),
      }
    case 'DECLINED':
      return {
        type: 'PaymentRejected',
        paymentId: ext.txn_id,
        rejectedAt: new Date(ext.ts * 1000),
      }
    case 'PENDING':
      return null
  }
}
```

The ACL lives in the infrastructure layer. The domain layer never sees `ExternalPaymentEvent`.

## Saga — Multi-Aggregate Coordination

### Order Fulfillment Saga

```typescript
type OrderFulfillmentSaga = {
  process: (event: DomainEvent) => Command[]
}

type Command =
  | { type: 'ReserveInventory'; orderId: string; items: OrderItem[] }
  | { type: 'ChargePayment'; orderId: string; amount: Money }
  | { type: 'ShipOrder'; orderId: string; address: Address }
  | { type: 'ReleaseInventory'; orderId: string; items: OrderItem[] }
  | { type: 'RefundPayment'; orderId: string; amount: Money }
  | { type: 'CancelOrder'; orderId: string; reason: string }

const orderFulfillmentSaga: OrderFulfillmentSaga = {
  process: (event) => {
    switch (event.type) {
      case 'OrderPlaced':
        return [{ type: 'ReserveInventory', orderId: event.orderId, items: event.items }]

      case 'InventoryReserved':
        return [{ type: 'ChargePayment', orderId: event.orderId, amount: event.totalAmount }]

      case 'PaymentCharged':
        return [{ type: 'ShipOrder', orderId: event.orderId, address: event.shippingAddress }]

      case 'InventoryReservationFailed':
        return [{ type: 'CancelOrder', orderId: event.orderId, reason: 'Out of stock' }]

      case 'PaymentFailed':
        return [
          { type: 'ReleaseInventory', orderId: event.orderId, items: event.items },
          { type: 'CancelOrder', orderId: event.orderId, reason: 'Payment failed' },
        ]

      case 'ShipmentFailed':
        return [
          { type: 'RefundPayment', orderId: event.orderId, amount: event.amount },
          { type: 'ReleaseInventory', orderId: event.orderId, items: event.items },
          { type: 'CancelOrder', orderId: event.orderId, reason: 'Shipment failed' },
        ]

      default:
        return []
    }
  },
}
```

Each compensating action undoes the effect of a previous step. Compensating actions are domain-specific (not generic rollbacks).

## Ports & Adapters — Project Structure

```
src/
├── domain/                    # No external dependencies
│   ├── ticket/
│   │   ├── ticket.ts          # Aggregate type + commands
│   │   ├── events.ts          # Domain events
│   │   └── repository.ts      # Port (interface only)
│   ├── shared/
│   │   ├── money.ts           # Value object
│   │   └── email.ts           # Value object
│   └── services/
│       └── pricing.ts         # Domain service
├── application/               # Use cases, orchestration
│   ├── escalate-ticket.ts
│   └── open-ticket.ts
├── infrastructure/            # Adapters (implementations)
│   ├── persistence/
│   │   └── postgres-ticket-repo.ts
│   ├── messaging/
│   │   └── rabbitmq-publisher.ts
│   └── external/
│       └── payment-gateway-acl.ts
└── presentation/              # HTTP handlers, CLI, event consumers
    ├── rest/
    │   └── ticket-controller.ts
    └── consumers/
        └── payment-event-consumer.ts
```

**Dependency rule:** domain imports nothing. Application imports domain. Infrastructure imports domain (implements ports). Presentation imports application.

## Value Object Patterns

### Money (Avoiding Floating Point Bugs)

```typescript
type Currency = 'USD' | 'EUR' | 'GBP'

type Money = {
  readonly cents: number
  readonly currency: Currency
}

const Money = (amount: number, currency: Currency): Money => ({
  cents: Math.round(amount * 100),
  currency,
})

const add = (a: Money, b: Money): Money => {
  if (a.currency !== b.currency) throw new Error(`Cannot add ${a.currency} to ${b.currency}`)
  return { cents: a.cents + b.cents, currency: a.currency }
}

const multiply = (m: Money, factor: number): Money => ({
  cents: Math.round(m.cents * factor),
  currency: m.currency,
})

const format = (m: Money): string => `${(m.cents / 100).toFixed(2)} ${m.currency}`
```

### DateRange (Enforcing Invariants)

```typescript
type DateRange = {
  readonly start: Date
  readonly end: Date
}

const DateRange = (start: Date, end: Date): DateRange => {
  if (end <= start) throw new Error('End must be after start')
  return { start, end }
}

const overlaps = (a: DateRange, b: DateRange): boolean =>
  a.start < b.end && b.start < a.end

const contains = (range: DateRange, date: Date): boolean =>
  date >= range.start && date <= range.end

const durationMs = (range: DateRange): number =>
  range.end.getTime() - range.start.getTime()
```

## Aggregate Design Checklist

1. **Is there an invariant?** If business rules span two objects, they belong in the same aggregate.
2. **Is strong consistency required?** If eventual consistency is acceptable, keep objects in separate aggregates.
3. **Is the aggregate small enough?** Large aggregates cause contention. Only include data needed for invariant enforcement.
4. **Does the root control all access?** External code never reaches past the root to internal entities.
5. **Are external aggregates referenced by ID only?** Never hold object references to other aggregates.
6. **Is there a version field for optimistic concurrency?** Every aggregate mutation must increment and check the version.
7. **Do domain events carry enough data?** Consumers should not need to query the source aggregate.
