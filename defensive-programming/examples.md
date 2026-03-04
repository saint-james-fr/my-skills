# Defensive Programming — Worked Examples

Complete, copy-pasteable examples organized by pattern. Each example shows the naive approach and the defensive alternative.

---

## Example 1: Boundary Parsing (TypeScript + Zod)

A REST endpoint that accepts user registration. Parse at the boundary, work with trusted types everywhere else.

```typescript
import { z } from "zod";

const RegistrationSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(18).max(120),
  username: z.string().min(3).max(64).regex(/^[a-z0-9_-]+$/),
});

type Registration = z.infer<typeof RegistrationSchema>;

// Handler — the only place that touches raw input
const handleRegister = async (req: Request): Promise<Response> => {
  const result = RegistrationSchema.safeParse(await req.json());

  if (!result.success) {
    return Response.json(
      { errors: result.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  // Everything downstream receives `Registration` — guaranteed valid
  await createUser(result.data);
  return Response.json({ ok: true }, { status: 201 });
};

const createUser = async (reg: Registration) => {
  // No validation needed — the type proves it
};
```

---

## Example 2: Branded Types Prevent ID Swap (TypeScript)

Two IDs that are both `number` at runtime but cannot be confused at compile time.

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type OrderId = Brand<number, "OrderId">;

const toUserId = (n: number): UserId => n as UserId;
const toOrderId = (n: number): OrderId => n as OrderId;

const getOrder = (userId: UserId, orderId: OrderId) => { /* ... */ };

const uid = toUserId(1);
const oid = toOrderId(42);

getOrder(uid, oid); // compiles
getOrder(oid, uid); // compile error — OrderId is not UserId
```

---

## Example 3: Discriminated Union State Machine (TypeScript)

A payment flow where only valid transitions compile.

```typescript
type PaymentState =
  | { type: "Pending" }
  | { type: "Authorized"; authCode: string }
  | { type: "Captured"; captureId: string }
  | { type: "Refunded"; refundId: string }
  | { type: "Failed"; reason: string };

const TRANSITIONS: Record<PaymentState["type"], PaymentState["type"][]> = {
  Pending: ["Authorized", "Failed"],
  Authorized: ["Captured", "Failed"],
  Captured: ["Refunded"],
  Refunded: [],
  Failed: [],
};

const assertNever = (x: never): never => {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
};

const transitionPayment = (
  from: PaymentState,
  to: PaymentState
): PaymentState => {
  const allowed = TRANSITIONS[from.type];
  if (!allowed.includes(to.type)) {
    throw new Error(`Illegal: ${from.type} → ${to.type}`);
  }
  return to;
};

const describePayment = (s: PaymentState): string => {
  switch (s.type) {
    case "Pending": return "Awaiting authorization";
    case "Authorized": return `Authorized: ${s.authCode}`;
    case "Captured": return `Captured: ${s.captureId}`;
    case "Refunded": return `Refunded: ${s.refundId}`;
    case "Failed": return `Failed: ${s.reason}`;
    default: return assertNever(s);
  }
};
```

---

## Example 4: Newtype with Validation (Rust)

A `Username` that can only be created if constraints are met.

```rust
pub struct Username(String);

impl Username {
    pub fn new(raw: impl Into<String>) -> Result<Self, &'static str> {
        let s = raw.into();
        if s.len() < 3 {
            return Err("too short");
        }
        if s.len() > 64 {
            return Err("too long");
        }
        if !s.chars().all(|c| c.is_ascii_alphanumeric() || c == '-' || c == '_') {
            return Err("invalid characters");
        }
        Ok(Self(s))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl TryFrom<&str> for Username {
    type Error = &'static str;
    fn try_from(value: &str) -> Result<Self, Self::Error> {
        Self::new(value)
    }
}
```

---

## Example 5: Pydantic Boundary Parsing (Python)

Parse external API responses into validated domain types.

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime

class SensorReading(BaseModel):
    sensor_id: str = Field(min_length=1)
    value: float = Field(ge=-100.0, le=200.0)
    timestamp: datetime

    @field_validator("timestamp")
    @classmethod
    def must_not_be_future(cls, v: datetime) -> datetime:
        if v > datetime.now():
            raise ValueError("Timestamp cannot be in the future")
        return v

    @property
    def age_ms(self) -> float:
        return (datetime.now() - self.timestamp).total_seconds() * 1000

    def require_fresh(self, max_age_ms: float = 500) -> "SensorReading":
        if self.age_ms > max_age_ms:
            raise ValueError(f"Reading is {self.age_ms:.0f}ms old (max {max_age_ms}ms)")
        return self

# At the boundary
raw = {"sensor_id": "temp-01", "value": 22.5, "timestamp": "2026-03-04T10:00:00"}
reading = SensorReading.model_validate(raw)  # parse, don't validate
fresh = reading.require_fresh(max_age_ms=1000)
```

---

## Example 6: Timeout + Abort (TypeScript)

Wrap any async operation with a deadline. Uses `AbortController` — no external deps.

```typescript
const withTimeout = async <T>(
  fn: (signal: AbortSignal) => Promise<T>,
  ms: number
): Promise<T> => {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);

  try {
    return await fn(controller.signal);
  } finally {
    clearTimeout(timer);
  }
};

// Usage
const data = await withTimeout(
  (signal) => fetch("https://api.example.com/data", { signal }).then((r) => r.json()),
  3000
);
```

---

## Example 7: Single-Flight Lock (TypeScript)

Prevent overlapping executions of a critical operation.

```typescript
class SingleFlight<T> {
  private inflight: Promise<T> | null = null;

  async run(fn: () => Promise<T>): Promise<T> {
    if (this.inflight) {
      return this.inflight; // deduplicate — return the in-progress result
    }

    this.inflight = fn().finally(() => {
      this.inflight = null;
    });

    return this.inflight;
  }
}

// Usage — concurrent callers share the same request
const flight = new SingleFlight<Response>();
const result = await flight.run(() => fetch("/api/expensive"));
```

---

## Example 8: Circuit Breaker (TypeScript)

Minimal circuit breaker for a remote dependency.

```typescript
type BreakerState = "closed" | "open" | "half-open";

class CircuitBreaker {
  private state: BreakerState = "closed";
  private failures = 0;
  private lastFailure = 0;

  constructor(
    private readonly threshold: number,
    private readonly cooldownMs: number
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailure > this.cooldownMs) {
        this.state = "half-open";
      } else {
        throw new Error("Circuit open — failing fast");
      }
    }

    try {
      const result = await fn();
      this.reset();
      return result;
    } catch (err) {
      this.recordFailure();
      throw err;
    }
  }

  private recordFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.threshold) {
      this.state = "open";
    }
  }

  private reset() {
    this.failures = 0;
    this.state = "closed";
  }
}
```

---

## Example 9: Exhaustive Match (Python 3.10+)

Using `match` with mypy strict mode to catch missing cases.

```python
from dataclasses import dataclass
from typing import Never

@dataclass(frozen=True)
class Idle:
    pass

@dataclass(frozen=True)
class Processing:
    started_at: float

@dataclass(frozen=True)
class Done:
    result: str

@dataclass(frozen=True)
class Failed:
    error: str

State = Idle | Processing | Done | Failed

def assert_never(x: Never) -> Never:
    raise AssertionError(f"Unhandled: {x}")

def describe(state: State) -> str:
    match state:
        case Idle():
            return "Idle"
        case Processing(started_at=t):
            return f"Processing since {t}"
        case Done(result=r):
            return f"Done: {r}"
        case Failed(error=e):
            return f"Failed: {e}"
        case _ as unreachable:
            assert_never(unreachable)  # mypy error if a variant is missing
```

---

## Example 10: Unexported Constructor (Go)

Force callers through a validating constructor.

```go
package money

import "errors"

type Amount struct {
	cents int64
}

func NewAmount(cents int64) (Amount, error) {
	if cents < 0 {
		return Amount{}, errors.New("amount cannot be negative")
	}
	if cents > 100_000_00 { // $100,000 cap
		return Amount{}, errors.New("amount exceeds limit")
	}
	return Amount{cents: cents}, nil
}

func (a Amount) Cents() int64 { return a.cents }

func (a Amount) Add(b Amount) (Amount, error) {
	return NewAmount(a.cents + b.cents) // re-validates on every operation
}
```

---

## Example 11: Watchdog / Deadman (TypeScript)

Periodic heartbeat check — triggers emergency action if heartbeat stops.

```typescript
class Watchdog {
  private timer: ReturnType<typeof setInterval>;
  private lastBeat = Date.now();

  constructor(
    private readonly intervalMs: number,
    private readonly onTimeout: () => void
  ) {
    this.timer = setInterval(() => this.check(), intervalMs);
  }

  beat() {
    this.lastBeat = Date.now();
  }

  private check() {
    if (Date.now() - this.lastBeat > this.intervalMs * 2) {
      this.onTimeout();
    }
  }

  stop() {
    clearInterval(this.timer);
  }
}

// Usage
const dog = new Watchdog(200, () => {
  console.error("No heartbeat — triggering safe shutdown");
  process.exit(1);
});

setInterval(() => dog.beat(), 100); // healthy heartbeat
```

---

## Example 12: Structured Concurrency (Go)

Tie goroutine lifetimes to a parent context. Cancellation cascades automatically.

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/sync/errgroup"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	g, ctx := errgroup.WithContext(ctx)

	g.Go(func() error { return pollSensor(ctx, "temp") })
	g.Go(func() error { return pollSensor(ctx, "pressure") })

	if err := g.Wait(); err != nil {
		fmt.Println("error:", err) // one failure cancels the group
	}
}

func pollSensor(ctx context.Context, name string) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(500 * time.Millisecond):
			fmt.Printf("[%s] reading\n", name)
		}
	}
}
```

---

## Example 13: Conditional Field Dependencies via Union (TypeScript)

From Javier Casas — when `field2` may only exist if `field1` exists, model the valid combos explicitly.

```typescript
type NoFields = { [key: string]: never };

type Field1Only = { field1: string };

type BothFields = { field1: string; field2: string };

type Fields = NoFields | Field1Only | BothFields;

const process = (f: Fields) => {
  if ("field1" in f) {
    if ("field2" in f) {
      return `${f.field1} + ${f.field2}`;
    }
    return f.field1;
  }
  return "empty";
};

// Compile error — field2 without field1 is not a valid Fields
// process({ field2: "oops" });
```

---

## Example 14: Contact Info — Making "At Least One" Unbreakable (TypeScript)

From Chris Krycho / Scott Wlaschin — a contact must have email, postal address, or both. "Neither" is unrepresentable.

```typescript
class EmailInfo {
  private constructor(public readonly email: string) {}

  static create(raw: string): EmailInfo | undefined {
    return /^\S+@\S+\.\S+$/.test(raw) ? new EmailInfo(raw) : undefined;
  }
}

class PostalInfo {
  constructor(public readonly address: string) {}
}

type ContactInfo =
  | { type: "email"; info: EmailInfo }
  | { type: "postal"; info: PostalInfo }
  | { type: "both"; email: EmailInfo; postal: PostalInfo };

type Contact = {
  readonly name: string;
  readonly contactInfo: ContactInfo;
};

const describe = (c: Contact): string => {
  switch (c.contactInfo.type) {
    case "email":
      return `${c.name} — email: ${c.contactInfo.info.email}`;
    case "postal":
      return `${c.name} — address: ${c.contactInfo.info.address}`;
    case "both":
      return `${c.name} — ${c.contactInfo.email.email}, ${c.contactInfo.postal.address}`;
  }
};
```

---

## Example 15: Strengthen Arguments, Don't Weaken Returns (TypeScript)

From Alexis King — instead of making `head()` return `T | undefined`, require a `NonEmpty` input.

```typescript
type NonEmpty<T> = [T, ...T[]];

// Parse once at the boundary
const parseNonEmpty = <T>(arr: T[]): NonEmpty<T> => {
  if (arr.length === 0) throw new Error("Array cannot be empty");
  return arr as NonEmpty<T>;
};

// These functions never need to check for emptiness
const head = <T>(arr: NonEmpty<T>): T => arr[0];
const last = <T>(arr: NonEmpty<T>): T => arr[arr.length - 1];

// Usage — check once, use everywhere
const dirs = parseNonEmpty(process.env.CONFIG_DIRS?.split(",") ?? []);
const primary = head(dirs); // always defined, no `| undefined`
```

---

## Example 16: Event-Driven State Machine with Exhaustive Transitions (TypeScript)

From OneUptime — model `(State, Event) → State` as a pure reducer. Invalid events for a given state are ignored (the machine is total).

```typescript
type State =
  | { type: "idle" }
  | { type: "loading"; startedAt: number }
  | { type: "success"; data: string; loadedAt: number }
  | { type: "error"; error: Error; retryCount: number };

type Event =
  | { type: "FETCH" }
  | { type: "SUCCESS"; data: string }
  | { type: "ERROR"; error: Error }
  | { type: "RETRY" }
  | { type: "RESET" };

const assertNever = (x: never): never => {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
};

const reduce = (state: State, event: Event): State => {
  switch (state.type) {
    case "idle":
      if (event.type === "FETCH") {
        return { type: "loading", startedAt: Date.now() };
      }
      return state;

    case "loading":
      if (event.type === "SUCCESS") {
        return { type: "success", data: event.data, loadedAt: Date.now() };
      }
      if (event.type === "ERROR") {
        return { type: "error", error: event.error, retryCount: 0 };
      }
      return state;

    case "success":
      if (event.type === "RESET") return { type: "idle" };
      return state;

    case "error":
      if (event.type === "RETRY") {
        return { type: "loading", startedAt: Date.now() };
      }
      if (event.type === "RESET") return { type: "idle" };
      return state;

    default:
      return assertNever(state);
  }
};
```

---

## Example 17: Fail Closed — Transaction Rollback (TypeScript)

From OWASP A10:2025 — never attempt to recover a transaction partway through. Roll back everything and start again.

```typescript
type TransactionStep =
  | { type: "debit"; account: string; amount: number }
  | { type: "credit"; account: string; amount: number }
  | { type: "log"; entry: string };

const executeTransaction = async (steps: TransactionStep[]) => {
  const completed: TransactionStep[] = [];

  try {
    for (const step of steps) {
      await executeStep(step);
      completed.push(step);
    }
  } catch (err) {
    // Fail closed: undo everything in reverse order
    for (const step of completed.reverse()) {
      await rollbackStep(step);
    }
    throw new Error(`Transaction failed and rolled back: ${err}`);
  }
};

const executeStep = async (step: TransactionStep) => { /* ... */ };

const rollbackStep = async (step: TransactionStep) => {
  switch (step.type) {
    case "debit":
      await executeStep({ type: "credit", account: step.account, amount: step.amount });
      break;
    case "credit":
      await executeStep({ type: "debit", account: step.account, amount: step.amount });
      break;
    case "log":
      break; // logs don't need rollback
  }
};
```
