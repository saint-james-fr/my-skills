# Defensive Programming Reference

Detailed patterns, anti-patterns, and further reading. Read this file when you need deeper guidance on a specific defensive technique.

---

## 1. Parse, Don't Validate

**Source**: [Alexis King (2019)](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — [TypeScript edition](https://bardeworks.com/blog/parse-dont-validate-typescript/)

### The problem with validation

Validation checks a property and returns a boolean, discarding the proof. Every downstream consumer must re-validate or trust on faith.

```typescript
// Validation: proof is thrown away
const isValidEmail = (s: string): boolean => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s);

function send(email: string) {
  if (!isValidEmail(email)) throw new Error("bad email");
  // email is still `string` — compiler learned nothing
}
```

### The parsing alternative

Parsing transforms unstructured data into a stricter type. The type carries the proof.

```typescript
type Email = string & { readonly __brand: "Email" };

const parseEmail = (s: string): Email => {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s)) throw new Error("Invalid email");
  return s as Email;
};

// Downstream functions accept `Email` — no re-validation needed
const send = (email: Email) => { /* guaranteed valid */ };
```

### Strengthen arguments, don't weaken returns

From Alexis King's original post: there are two ways to turn a partial function total.

1. **Weaken the return type** — return `Maybe` / `T | undefined`. This pushes the burden onto every caller, who must handle the empty case even when they *know* it won't happen.
2. **Strengthen the argument type** — accept `NonEmpty<T>` instead of `T[]`. This eliminates the impossible case at the call site. The check happens once, at the boundary.

Always prefer option 2. It moves the proof upward (toward the boundary) rather than downward (into every consumer).

```typescript
// Weaken return (callers must handle undefined everywhere)
const head = <T>(arr: T[]): T | undefined => arr[0];

// Strengthen argument (callers must prove non-emptiness once)
type NonEmpty<T> = [T, ...T[]];
const head = <T>(arr: NonEmpty<T>): T => arr[0];
```

### The shotgun parsing anti-pattern

From language-theoretic security research:

> **Shotgun parsing** is a programming antipattern whereby parsing and input-validating code is mixed with and spread across processing code — throwing a cloud of checks at the input, and hoping, without any systematic justification, that one or another would catch all the "bad" cases.

The problem: if you discover invalid data *after* you've already acted on part of the input, you may need to roll back side effects. Parsing up front splits the program into two phases — parse then execute — so failure from bad input can only happen in the first phase.

### Practical rules (from Alexis King)

| Rule | Why |
|------|-----|
| Focus on the datatypes — write functions on the representation you *wish* you had | Design becomes an exercise in bridging the gap |
| Push the burden of proof upward as far as possible, but no further | Parse at the boundary, before any data is acted upon |
| Avoid denormalized representations, especially mutable ones | Duplicated data = trivially representable illegal state (things getting out of sync) |
| Treat functions returning `void` / `()` with suspicion | If the primary purpose is raising an error, there's likely a better design that returns a refined type |
| Don't be afraid to parse in multiple passes | Avoiding shotgun parsing means don't *act* before fully parsing — not that you can't use partial results to decide how to parse the rest |

### Library support

| Language | Library | Notes |
|----------|---------|-------|
| TypeScript | Zod, Valibot, ArkType, io-ts | Zod is most popular; Valibot for smaller bundles |
| Python | Pydantic, attrs + cattrs | Pydantic v2 is Rust-backed, very fast |
| Rust | serde, validator crate | `TryFrom` trait for custom newtypes |
| Go | go-playground/validator, ozzo-validation | Often combined with custom constructor functions |

---

## 2. Make Illegal States Unrepresentable

**Source**: [corrode.dev](https://corrode.dev/blog/illegal-state/) — [Yuri Bogomolov](https://ybogomolov.me/making-illegal-states-unrepresentable) — [Chris Krycho](https://v5.chriskrycho.com/journal/making-illegal-states-unrepresentable-in-ts/)

### Core idea

If the type system allows constructing a value that violates business rules, a bug is inevitable. Restructure types so the invalid combination cannot be expressed. As Scott Wlaschin puts it: enumerate the valid cases explicitly, and the compiler enforces them for you.

### The contact info example (from Chris Krycho / Scott Wlaschin)

Business rule: "A contact must have an email *or* a postal address (or both)."

```typescript
// BAD: optional fields allow having neither
type Contact = {
  name: string;
  email?: string;
  address?: string;
};

// GOOD: union of valid combinations only
type ContactInfo =
  | { type: "email"; email: Email }
  | { type: "postal"; address: Address }
  | { type: "both"; email: Email; address: Address };

type Contact = { name: Name; info: ContactInfo };
```

Now a `Contact` with no contact info is unrepresentable. Adding a new business rule (e.g. "phone only") means adding a variant — and every `switch` immediately breaks if it doesn't handle it.

### Conditional field dependencies (from Javier Casas)

When `field2` may only exist if `field1` exists, don't use two optional fields. Enumerate the valid combinations as a union.

```typescript
// BAD: field2 without field1 is representable
type Fields = { field1?: string; field2?: string };

// GOOD: explicitly enumerate valid combos
type Fields =
  | { [key: string]: never }           // no fields
  | { field1: string }                 // field1 only
  | { field1: string; field2: string } // both
```

The `{ [key: string]: never }` trick bans unexpected fields in TypeScript's structural type system — since no value satisfies `never`, the only valid object is `{}`.

### Structural typing traps (TypeScript)

TypeScript uses structural typing, which means a value with *extra* fields often passes type checks. Be aware of these bypasses:

- Assigning through an intermediate `{}` or `any` variable defeats the check.
- Direct object literals get "excess property checking" — but variables don't.
- Use `private constructor` + static `create()` to enforce validation even when structural typing would otherwise allow construction.

### Techniques by language

**TypeScript**
- **Branded types**: `type UserId = number & { __brand: "UserId" }` — prevents swapping `UserId` and `OrderId`.
- **Discriminated unions**: Model each state as a separate branch with a literal `type` tag. The compiler narrows automatically in `switch`.
- **Tuple types for non-empty**: `type NonEmpty<T> = [T, ...T[]]` — `arr[0]` is always safe.
- **Private constructors (class)**: `private constructor` + static `create()` method. The only construction path goes through your parser.
- **Tagged unions with literal `type` field**: Each branch gets a `readonly type = "Variant" as const` property. Use `switch` on `type` for exhaustive handling with `assertNever` in the `default` branch.

**Rust**
- **Newtype pattern**: `struct Username(String)` with a validating `fn new() -> Result<Self, E>`. Keep the inner field private so only the constructor can create it.
- **Enums for state machines**: Each variant carries only the data valid for that state. `match` is exhaustive by default.
- **`TryFrom` trait**: Ergonomic conversion with automatic validation.
- **Module privacy**: `pub struct Foo(String)` in a module — the tuple field is private to external consumers. Cannot be bypassed from outside the module.

**Python**
- **`NewType`**: `UserId = NewType("UserId", int)` — lightweight nominal wrapper.
- **Frozen dataclasses**: `@dataclass(frozen=True)` prevents mutation after construction.
- **`__init__` / `__post_init__` validation**: Raise `ValueError` to reject invalid construction.
- **Pydantic models**: Declarative field constraints with automatic validation on construction.

**Go**
- **Unexported struct + exported constructor**: `type username struct{ v string }` + `func NewUsername(s string) (username, error)`.
- **Interface satisfaction**: Require types to implement a marker interface rather than passing raw strings.

### Anti-pattern: optional fields that create impossible combinations

```typescript
// BAD: nothing prevents both being set or both being absent
type Response = {
  data?: string;
  error?: string;
};

// GOOD: exactly one is always present
type Response =
  | { type: "success"; data: string }
  | { type: "error"; error: string };
```

### Anti-pattern: boolean flags for state

```typescript
// BAD: allows { isLoading: true, isError: true, data: "hello", error: Error }
type DataState = {
  isLoading: boolean;
  isError: boolean;
  data: string | null;
  error: Error | null;
};

// GOOD: each state carries only its valid data
type DataState =
  | { type: "idle" }
  | { type: "loading"; startedAt: number }
  | { type: "success"; data: string }
  | { type: "error"; error: Error; retryCount: number };
```

---

## 3. Exhaustive State Handling

### The `assertNever` pattern (TypeScript)

```typescript
const assertNever = (x: never): never => {
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`);
};

type State = { type: "Idle" } | { type: "Loading" } | { type: "Done"; result: string };

const label = (s: State): string => {
  switch (s.type) {
    case "Idle": return "Waiting";
    case "Loading": return "In progress";
    case "Done": return s.result;
    default: return assertNever(s); // compile error if a variant is missing
  }
};
```

Adding a new variant to `State` immediately breaks every `switch` that doesn't handle it.

### Equivalent in other languages

| Language | Mechanism |
|----------|-----------|
| Rust | `match` is exhaustive by default; `#[non_exhaustive]` for public enums |
| Python 3.10+ | `match` + type checkers (mypy `--strict`) flag missing cases |
| Go | No native exhaustive switch; use linters like `exhaustive` or code generation |

---

## 4. Concurrency Hazards & Guards

### Common hazards

| Hazard | Description | Typical symptom |
|--------|-------------|-----------------|
| Data race | Two threads read/write shared state without synchronization | Corrupted data, intermittent crashes |
| Race condition | Outcome depends on thread scheduling order | Non-deterministic behavior |
| Deadlock | Two locks acquired in different order across threads | System hangs |
| Starvation | A thread never gets scheduled | Feature silently stops working |
| Lost update | Read-modify-write without atomicity | Counter drift, missing writes |

### Defensive patterns

**Single-flight / motion lock**: Only one operation of a kind runs at a time. Reject or queue duplicates.

```typescript
class SingleFlight {
  private inflight = false;

  async run<T>(fn: () => Promise<T>): Promise<T> {
    if (this.inflight) throw new Error("Already in flight");
    this.inflight = true;
    try { return await fn(); }
    finally { this.inflight = false; }
  }
}
```

**Guarded suspension**: A thread waits for a precondition before proceeding, rather than polling.

**Actor model**: Isolate mutable state inside an actor that processes messages sequentially. No shared state, no locks.

**Structured concurrency**: Tie the lifetime of spawned tasks to a parent scope. When the parent cancels, all children cancel. Prevents leaked goroutines / dangling promises.

---

## 5. Runtime Guards

### Timeouts

Every external call needs a deadline. Without one, a hung dependency blocks the caller forever.

```typescript
const withTimeout = <T>(fn: () => Promise<T>, ms: number): Promise<T> => {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);
  return fn().finally(() => clearTimeout(timer));
};
```

### Circuit breaker

Prevent cascading failures by failing fast when a dependency is unhealthy. From Michael Nygard's *Release It!*, popularized by [Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html).

| State | Behavior |
|-------|----------|
| **Closed** | Requests pass through; failures are counted. Successful calls reset the counter. |
| **Open** | Requests fail immediately without calling the dependency. After a `resetTimeout` period, transitions to half-open. |
| **Half-open** | A single probe request tests recovery. Success → closed. Failure → back to open. |

**Key design considerations** (from Fowler):
- Not all errors should trip the breaker — distinguish transient failures from permanent ones.
- Pair with a **thread pool / connection limit** so the breaker trips when resources are exhausted, not just on timeouts.
- Circuit breakers are a valuable **monitoring** point — any state change should be logged and alertable.
- Callers must decide what to do when the breaker is open: queue for later, return cached/stale data, or show a degraded experience.

### Watchdog / deadman switch

A periodic heartbeat must arrive within a window. If it stops, assume the system is unhealthy and trigger a safe shutdown or alert.

```typescript
class Deadman {
  private last = Date.now();
  constructor(private readonly maxSilenceMs: number) {}

  signal() { this.last = Date.now(); }

  check() {
    if (Date.now() - this.last > this.maxSilenceMs) {
      throw new Error("Deadman triggered — no heartbeat received");
    }
  }
}
```

### Stale data rejection

Time-sensitive reads must check freshness. Stale sensor data or cached values can cause incorrect decisions.

```typescript
type Timestamped<T> = { value: T; at: number };

const requireFresh = <T>(data: Timestamped<T>, maxAgeMs: number): T => {
  if (Date.now() - data.at > maxAgeMs) throw new Error("Data stale");
  return data.value;
};
```

---

## 6. Guard Clauses & Fail Fast

Exit early from invalid states. This flattens code, reduces nesting, and surfaces bugs immediately.

```typescript
const processOrder = (order: Order) => {
  if (!order.items.length) throw new Error("Empty order");
  if (order.total <= 0) throw new Error("Invalid total");
  if (!order.customer) throw new Error("Missing customer");

  // happy path — no nesting
  return submitOrder(order);
};
```

### Assertion functions (TypeScript)

```typescript
function assertDefined<T>(val: T | undefined, msg: string): asserts val is T {
  if (val === undefined) throw new Error(msg);
}

const user = getUser(id);
assertDefined(user, `User ${id} not found`);
// TypeScript now knows `user` is defined
```

---

## 7. Error Handling Hierarchy

Not all errors are equal. Defensive code classifies errors and responds proportionally.

| Category | Example | Response |
|----------|---------|----------|
| **Programmer error** | Null dereference, out-of-bounds | Crash + log (fix the code) |
| **Operational error** | Network timeout, disk full | Retry / degrade / alert |
| **Business rule violation** | Insufficient funds | Return error to caller |
| **Security violation** | Auth failure, tampered input | Reject + audit log |

### OWASP Top 10:2025 A10 — Mishandling Exceptional Conditions

This is now a top-10 category (24 mapped CWEs). Key failure modes:

| CWE | What goes wrong |
|-----|-----------------|
| CWE-636 | **Failing open** — system grants access or proceeds when it should deny |
| CWE-209 | **Leaking sensitive info in errors** — stack traces, SQL, credentials exposed to users |
| CWE-460 | **Improper cleanup on exception** — resources locked, connections leaked |
| CWE-252 | **Unchecked return value** — error silently ignored |
| CWE-478 | **Missing default case in switch** — unhandled variant falls through silently |

**OWASP prevention guidance**:
- Catch errors *at the function where they occur*, not at a high level.
- Always **fail closed**: roll back the entire transaction, don't try to recover partway through.
- Use **rate limiting and resource quotas** to prevent exceptional conditions in the first place.
- Implement a **global exception handler** as a safety net.
- Never expose raw error details to clients — sanitize before returning.
- Monitor for repeated errors or patterns that indicate an ongoing attack.

---

## 8. State Machine Design

### When to use a state machine

- The system has distinct modes of operation.
- Certain actions are only valid in certain modes.
- Transitions between modes have preconditions.

### Design checklist

- [ ] Every state is explicitly named (no implicit "between" states).
- [ ] Every transition has a defined trigger and guard condition.
- [ ] Forbidden transitions are documented (and enforced).
- [ ] Entry/exit actions are defined (e.g., start timer on enter, cancel on exit).
- [ ] A "stuck" state is impossible — every state has at least one outbound transition or is a terminal state.

### Transition enforcement (runtime lookup table)

```typescript
type State =
  | { type: "Idle" }
  | { type: "Processing"; startedAt: number }
  | { type: "Done"; result: string }
  | { type: "Failed"; error: string };

const ALLOWED_TRANSITIONS: Record<State["type"], State["type"][]> = {
  Idle: ["Processing"],
  Processing: ["Done", "Failed"],
  Done: ["Idle"],
  Failed: ["Idle"],
};

const transition = (from: State, to: State): State => {
  if (!ALLOWED_TRANSITIONS[from.type].includes(to.type)) {
    throw new Error(`Illegal transition: ${from.type} → ${to.type}`);
  }
  return to;
};
```

### Event-driven transitions (from OneUptime)

Model transitions as `(State, Event) → State` for richer control. Events carry the payload needed for the target state.

```typescript
type Event =
  | { type: "START" }
  | { type: "SUCCEED"; data: string }
  | { type: "FAIL"; error: Error }
  | { type: "RETRY" }
  | { type: "RESET" };

const reduce = (state: State, event: Event): State => {
  switch (state.type) {
    case "Idle":
      return event.type === "START"
        ? { type: "Processing", startedAt: Date.now() }
        : state;
    case "Processing":
      if (event.type === "SUCCEED") return { type: "Done", result: event.data };
      if (event.type === "FAIL") return { type: "Failed", error: event.error.message };
      return state;
    case "Done":
      return event.type === "RESET" ? { type: "Idle" } : state;
    case "Failed":
      if (event.type === "RETRY") return { type: "Processing", startedAt: Date.now() };
      if (event.type === "RESET") return { type: "Idle" };
      return state;
  }
};
```

This pattern is identical to a Redux reducer, an XState machine definition, or an Elm `update` function. The `switch` is exhaustive via `assertNever` in the `default` branch. Invalid events for a given state are silently ignored (return `state`) — an intentional design choice that keeps the machine total.

---

## 9. Recommended Reading (books)

| Book | Author | Why |
|------|--------|-----|
| *Release It!* | Michael Nygard | Stability patterns: circuit breaker, bulkhead, timeout, steady state. The canonical reference for production-grade defensive design. |
| *Domain Modeling Made Functional* | Scott Wlaschin | End-to-end guide to making illegal states unrepresentable using algebraic data types. Uses F# but the ideas are universal. |
| *The Seven Turrets of Babel* (paper) | Momot et al. | Language-theoretic security — defines and names shotgun parsing, explains why it leads to vulnerabilities. |

---

## Further Reading

| Topic | Resource |
|-------|----------|
| Parse, don't validate | [Alexis King (original, 2019)](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) |
| Parse, don't validate (TypeScript) | [Anuraag Barde](https://bardeworks.com/blog/parse-dont-validate-typescript/) |
| Designing with types (F#) | [Scott Wlaschin — F# for Fun and Profit](https://fsharpforfunandprofit.com/series/designing-with-types.html) |
| Making impossible states impossible (talk) | [Richard Feldman — Elm Conf 2016](https://www.youtube.com/watch?v=IcgmSRJHu_8) |
| Illegal states (Rust) | [corrode.dev](https://corrode.dev/blog/illegal-state/) |
| Illegal states (TypeScript) | [Javier Casas](https://www.javiercasas.com/articles/typescript-impossible-states-irrepresentable/) |
| Illegal states (TypeScript) | [Chris Krycho](https://v5.chriskrycho.com/journal/making-illegal-states-unrepresentable-in-ts/) |
| Branded types | [typescript.tv](https://typescript.tv/best-practices/improve-your-type-safety-with-branded-types/) |
| Type Safety Back and Forth | [Matt Parsons](https://www.parsonsmatt.org/2017/10/11/type_safety_back_and_forth.html) |
| Ghosts of Departed Proofs (paper) | [Matt Noonan (2018)](https://kataskeue.com/gdp.pdf) |
| Circuit breaker | [Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html) |
| Shotgun parsing (paper) | [The Seven Turrets of Babel (2016)](http://langsec.org/papers/langsec-cwes-secdev2016.pdf) |
| OWASP error handling | [OWASP Top 10:2025 A10](https://owasp.org/Top10/2025/A10_2025-Mishandling_of_Exceptional_Conditions/) |
| OWASP error handling cheat sheet | [OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html) |
| Type-safe state machines (TypeScript) | [oneuptime.com (2026)](https://oneuptime.com/blog/post/2026-01-30-typescript-type-safe-state-machines/view) |
| Guarded suspension pattern | [java-design-patterns.com](https://java-design-patterns.com/patterns/guarded-suspension/) |
| Concurrency without races (Go) | [Syarif / Medium](https://elsyarifx.medium.com/goroutine-patterns-how-senior-developers-handle-concurrency-without-race-conditions-b4bd7fe94c8c) |
