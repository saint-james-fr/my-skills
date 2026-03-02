---
name: ddd-clean-architecture
description: Domain-Driven Design and Clean Architecture patterns for structuring complex business applications. Use when designing bounded contexts, aggregates, value objects, domain services, repositories, or choosing between layered/hexagonal/CQRS architectures. Covers strategic design (subdomains, context mapping) and tactical design (building blocks, communication patterns).
---

# Domain-Driven Design & Clean Architecture

## Strategic Design

### Subdomains

Every business domain decomposes into subdomains. Classify each:

| Type           | What it is                                        | Investment | Implementation                             |
| -------------- | ------------------------------------------------- | ---------- | ------------------------------------------ |
| **Core**       | Competitive advantage — what you do differently   | High       | Domain Model or Event-Sourced Domain Model |
| **Supporting** | Necessary but no competitive edge                 | Medium     | Active Record or Transaction Script        |
| **Generic**    | Same across all companies (auth, email, payments) | Low        | Buy/integrate off-the-shelf                |

Subdomain types evolve over time. A core subdomain can become generic when competitors commoditize it (e.g., route optimization → DeliverIT as-a-service). A generic subdomain can become core when you invest in a custom solution (e.g., Amazon → AWS).

### Bounded Context

A bounded context is a linguistic boundary — a single ubiquitous language applies consistently within it. Same term can mean different things in different contexts (e.g., "Ticket" in support vs. billing).

**Sizing heuristic:** start with wider boundaries, especially around core subdomains. Refactoring logical boundaries is cheaper than physical ones. Decompose as domain knowledge deepens.

A bounded context can contain multiple subdomains. A subdomain should not span multiple bounded contexts.

### Context Map Patterns

Relationships between bounded contexts:

| Pattern                        | Power dynamic                              | When to use                                       |
| ------------------------------ | ------------------------------------------ | ------------------------------------------------- |
| **Partnership**                | Mutual, ad-hoc coordination                | Two teams that communicate well and co-evolve     |
| **Shared Kernel**              | Co-owned subset of the model               | Limited overlap that both teams maintain together |
| **Customer–Supplier**          | Downstream needs drive upstream priorities | Upstream serves downstream's requirements         |
| **Conformist**                 | Downstream conforms to upstream model      | No leverage to influence upstream                 |
| **Anticorruption Layer (ACL)** | Downstream translates upstream model       | Protect your model from external models           |
| **Open-Host Service (OHS)**    | Upstream exposes integration-specific API  | Decouple public contract from internal model      |
| **Separate Ways**              | No integration                             | Cost of integration exceeds benefit               |

### Ubiquitous Language

The shared language between developers and domain experts within a bounded context. It appears in code (class names, method names), conversations, and documentation. Inconsistency in language signals a missing or wrong boundary.

## Tactical Design — Building Blocks

### Value Objects

Objects identified by their values, not by an ID. Two instances with identical fields are interchangeable.

**Rules:**

- Immutable — operations return new instances
- Validate in constructor — invalid state is unrepresentable
- Implement equality by value
- Encapsulate domain logic that manipulates the values

```typescript
type Money = {
  readonly amount: number;
  readonly currency: string;
};

const addMoney = (a: Money, b: Money): Money => {
  if (a.currency !== b.currency) throw new Error("Currency mismatch");
  return { amount: a.amount + b.amount, currency: a.currency };
};
```

**Use for:** all properties of entities — email, phone, address, money, date ranges, coordinates, status enums. Eliminates primitive obsession.

### Entities

Objects with a unique identity that persists through state changes. Two entities can have identical attributes but different identities.

**Rules:**

- Define identity explicitly (ID field, immutable after creation)
- Keep entity definition focused on identity and lifecycle
- Push attributes and behavior into value objects where possible

Entities are never used standalone — only within aggregates.

### Aggregates

A cluster of entities and value objects treated as a single unit for data changes. The aggregate is a **consistency boundary** and a **transaction boundary**.

**Rules:**

1. One entity is the **aggregate root** — the only entry point for external access
2. External objects reference the aggregate by its root ID only
3. Internal entities have local identity (meaningful only within the aggregate)
4. All invariants within the boundary must be satisfied after each transaction
5. One aggregate per database transaction — never commit changes to multiple aggregates atomically
6. Keep aggregates small — only include what must be strongly consistent

```typescript
type Ticket = {
  readonly id: TicketId;
  version: number;
  status: TicketStatus;
  assignedAgent: AgentId; // reference by ID — external aggregate
  messages: Message[]; // owned entity — inside the boundary
  escalation: Escalation | null; // owned value object
};

const escalateTicket = (ticket: Ticket, reason: string): Ticket => {
  if (ticket.status === "closed") throw new Error("Cannot escalate closed ticket");
  return {
    ...ticket,
    version: ticket.version + 1,
    escalation: { reason, escalatedAt: new Date(), previousAgent: ticket.assignedAgent },
  };
};
```

**Concurrency:** use optimistic locking via version field. On save, assert `WHERE version = @expectedVersion`.

**Cross-aggregate consistency** is always eventual. Use domain events + outbox pattern to propagate changes.

### Domain Services

Stateless operations that don't belong to any entity or value object. Use when the operation spans multiple aggregates or doesn't conceptually fit a single object.

```typescript
const transferMoney = (from: Account, to: Account, amount: Money, repo: AccountRepository): void => {
  from.debit(amount);
  to.credit(amount);
  repo.save(from);
  repo.save(to);
};
```

Keep domain services thin. If a service grows, the missing concept is likely a new aggregate or value object.

### Domain Events

Something that happened in the domain that other parts of the system care about. Named in past tense using ubiquitous language.

```typescript
type TicketEscalated = {
  readonly type: "TicketEscalated";
  readonly ticketId: TicketId;
  readonly reason: string;
  readonly occurredAt: Date;
};
```

Events are the mechanism for cross-aggregate and cross-context communication.

### Repositories

Collection-like interfaces for retrieving and persisting aggregates. One repository per aggregate root.

```typescript
type TicketRepository = {
  load: (id: TicketId) => Promise<Ticket>;
  save: (ticket: Ticket) => Promise<void>;
};
```

The repository interface lives in the domain layer. The implementation lives in the infrastructure layer (ports & adapters).

### Factories

Encapsulate complex aggregate creation. Use when construction involves invariant enforcement, identity generation, or assembly of multiple entities.

Simple aggregates: constructor is fine. Complex aggregates: use a factory function or factory method on a related aggregate root.

## Architectural Patterns

### Decision Tree

```
Subdomain type?
├─ Core (complex business logic)
│  ├─ Needs audit log / monetary tracking / deep analytics?
│  │  ├─ Yes → Event-Sourced Domain Model + CQRS
│  │  └─ No  → Domain Model + Ports & Adapters
│  └─ Testing: Pyramid (unit-heavy)
├─ Supporting (no competitive edge)
│  ├─ Complex data structures?
│  │  ├─ Yes → Active Record + Layered + Service Layer
│  │  └─ No  → Transaction Script + Layered
│  └─ Testing: Diamond (integration-heavy) or Reversed Pyramid (e2e-heavy)
└─ Generic → Buy/integrate, don't build
```

### Layered Architecture

Top-down dependencies: Presentation → Application → Business Logic → Data Access.

Best fit: Transaction Script, Active Record. Business logic layer depends on data access layer.

```
┌─────────────────────────┐
│   Presentation (API)    │
├─────────────────────────┤
│   Application Layer     │  ← orchestration, transaction scripts
├─────────────────────────┤
│   Business Logic Layer  │  ← active records, domain logic
├─────────────────────────┤
│   Data Access Layer     │  ← DB, external APIs
└─────────────────────────┘
```

### Ports & Adapters (Hexagonal / Clean / Onion)

Business logic at the center, infrastructure depends on domain (dependency inversion).

Best fit: Domain Model. Aggregates and value objects have zero infrastructure dependencies.

```
┌──────────────────────────────────────┐
│         Infrastructure               │
│  ┌──────────────────────────────┐    │
│  │      Application Layer       │    │
│  │  ┌──────────────────────┐    │    │
│  │  │    Domain Layer      │    │    │
│  │  │  (Aggregates, VOs,   │    │    │
│  │  │   Domain Services)   │    │    │
│  │  └──────────────────────┘    │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

**Ports** = interfaces defined by domain layer (e.g., `TicketRepository`).
**Adapters** = concrete implementations in infrastructure (e.g., `PostgresTicketRepository`).

### CQRS

Separate command model (writes) from read models (queries). Command model is the source of truth with optimistic concurrency. Read models are projections — precached, eventually consistent, rebuildable from scratch.

Use when: event-sourced domain model (mandatory), or any system needing multiple representations of the same data.

Commands can and should return data (from the strongly consistent model). The myth that "commands return void" leads to bad UX.

## Communication Patterns

### Outbox Pattern

Reliable domain event publishing. Solves the dual-write problem (DB commit + message bus publish).

1. Commit aggregate state + new domain events in the **same DB transaction**
2. Message relay reads unpublished events from DB
3. Relay publishes to message bus
4. Relay marks events as published

Guarantees **at-least-once delivery**. Consumers must be idempotent.

### Saga

Coordinates a business process spanning multiple aggregates/contexts via event-to-command matching. Linear flow, often stateless.

```
Event A → Command B → Event C → Command D
```

Each step is eventually consistent. If a step fails, the saga issues **compensating actions** (not rollbacks — you can't undo a sent email).

### Process Manager

Like a saga but with branching logic (if-else), internal state, and explicit lifecycle. Implemented as an aggregate (state-based or event-sourced).

**Rule of thumb:** if your saga has `if` statements, it's a process manager.

## Key Principles

- **Model the domain, not the data.** Your persistence schema follows your domain model, not the reverse.
- **Aggregates are consistency boundaries, not entity groups.** Only include what must be strongly consistent within a single transaction.
- **Cross-aggregate = eventually consistent.** If you need to change two aggregates atomically, your boundaries are wrong.
- **Ubiquitous language is non-negotiable.** Code must speak the same language as domain experts. If a concept doesn't have a name in the code, it doesn't exist in the model.
- **Use the simplest pattern that works.** Transaction Script for simple logic. Domain Model only for genuinely complex business rules. Don't over-engineer supporting subdomains.

For detailed implementation examples (aggregate lifecycle, ACL implementation, outbox with TypeScript, saga orchestration), see [ddd-clean-architecture-detail.md](references/ddd-clean-architecture-detail.md).
