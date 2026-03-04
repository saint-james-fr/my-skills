---
name: defensive-programming
description: Generate defensive programming safety specs for stateful or safety-critical systems. Use when designing systems with concurrency, hardware interaction, real-time constraints, state machines, or when the user asks about invariants, illegal states, guard clauses, failure modes, or defensive coding patterns.
---

# Defensive Programming Spec Generator

Produce a structured safety specification document before any implementation begins. The spec captures invariants, failure modes, illegal states, and runtime guards so that the implementation is constrained by design.

## Workflow

### Phase 1 — Elicit the System Boundaries

Before writing anything, interview the user. Ask concise, pointed questions organized into these categories:

**Invariants** — What must always be true?

- What properties must hold at every point in execution?
- Are there value ranges that must never be exceeded?
- Are there ordering guarantees (e.g. init before use, acquire before release)?

**Illegal states** — What must never happen?

- Which combinations of state are logically impossible but representable in code? (e.g. boolean flags that allow `{ isLoading: true, isError: true }`)
- Are there conditional field dependencies? (field B only valid when field A is present)
- Can the type system make these unrepresentable? (discriminated unions, branded types, private constructors)
- Are there transitions that should be forbidden?

**Failure modes** — What goes wrong in the real world?

- Latency spikes, jitter, clock drift
- Stale or missing data from external sources
- Network partitions, partial failures
- Resource exhaustion (memory, file descriptors, connections)
- Hardware lies (sensors returning garbage, actuators not responding)

**Concurrency** — Where do things run in parallel?

- Shared mutable state across threads/workers/processes
- Race conditions on read-modify-write sequences
- Async cancellation and cleanup
- Ordering guarantees (or lack thereof) between concurrent operations

**External boundaries** — What crosses a trust boundary?

- User input, API payloads, file contents
- Third-party service responses
- Configuration values loaded at runtime
- Is parsing happening at the boundary or is validation scattered throughout? (shotgun parsing)

**Error handling** — How does the system respond to failure?

- Does it fail closed (deny/rollback) or fail open (permit/continue)?
- Are transactions atomic — all-or-nothing — or can partial state survive a failure?
- Are error details sanitized before reaching external consumers?

Ask only questions relevant to the system described. Skip categories that don't apply.

### Phase 2 — Produce the Safety Spec

Once you have answers, generate a markdown document with these sections:

```markdown
# Safety Spec: [System Name]

## 1. System Overview

One paragraph describing what the system does and why it is safety-critical or failure-sensitive.

## 2. Invariants

| ID     | Invariant                        | Enforcement                                 |
| ------ | -------------------------------- | ------------------------------------------- |
| INV-01 | [property that must always hold] | [how: type, assertion, guard, architecture] |

## 3. Illegal States

| ID     | Illegal State                               | Prevention                                    |
| ------ | ------------------------------------------- | --------------------------------------------- |
| ILL-01 | [state combination that must be impossible] | [how: type system, state machine, validation] |

## 4. Failure Modes & Guards

| ID    | Failure Mode        | Impact                     | Guard                                                   |
| ----- | ------------------- | -------------------------- | ------------------------------------------------------- |
| FM-01 | [what can go wrong] | [consequence if unhandled] | [mitigation: timeout, retry, circuit breaker, fallback] |

## 5. Concurrency Hazards

| ID    | Hazard                                       | Affected Components | Mitigation                                |
| ----- | -------------------------------------------- | ------------------- | ----------------------------------------- |
| CH-01 | [race condition / deadlock / ordering issue] | [where]             | [lock, single-flight, queue, actor model] |

## 6. Boundary Validation

| Boundary | Input  | Validation Rule                          |
| -------- | ------ | ---------------------------------------- |
| [source] | [data] | [check: parse, schema, range, freshness] |

## 7. State Machine (if applicable)

Define states, valid transitions, and forbidden transitions.

| From | To  | Condition | Forbidden? |
| ---- | --- | --------- | ---------- |

## 8. Defensive Patterns Checklist

- [ ] All external input is parsed (not just validated) at the boundary into refined types
- [ ] No shotgun parsing — validation is not scattered across processing code
- [ ] Timeouts on all blocking/async operations
- [ ] Stale data detection on time-sensitive reads
- [ ] No shared mutable state without synchronization
- [ ] Exhaustive handling of all state/type variants (assertNever / match)
- [ ] No boolean flags for mutually exclusive states — use discriminated unions
- [ ] Graceful degradation path for every failure mode
- [ ] Fail closed on error — transactions roll back entirely, never partially committed
- [ ] Error responses sanitized — no stack traces, SQL, or credentials leaked
- [ ] Logging/alerting for invariant violations (never silent)
```

### Phase 3 — Review Pass

After generating the spec, self-review with these questions:

- Is every invariant enforceable at compile time, startup, or runtime?
- Is every failure mode paired with a concrete guard, not just "handle errors"?
- Are concurrency hazards traced to specific shared resources?
- Does the state machine have any implicit or missing transitions?
- Is parsing happening at the boundary, or is validation scattered? (shotgun parsing)
- Are there boolean flag combinations that should be discriminated unions instead?
- Does the system fail closed on error, or could partial state survive a crash?
- Could any function's return type be strengthened to carry proof of validity?

Surface any gaps or ambiguities back to the user.

## Principles

- **Parse, don't validate.** Transform unstructured data into typed, constrained representations at the boundary. The type carries the proof — downstream code never re-validates. Avoid shotgun parsing: don't scatter checks across processing code. ([Alexis King](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/))
- **Strengthen arguments, don't weaken returns.** Prefer `head(arr: NonEmpty<T>): T` over `head(arr: T[]): T | undefined`. Move the proof upward toward the boundary, not downward into every consumer.
- **Make illegal states unrepresentable.** Prefer type-level constraints over runtime checks. Replace boolean flags with discriminated unions. Replace optional field combos with explicit variant types. ([corrode.dev](https://corrode.dev/blog/illegal-state/) — [Chris Krycho](https://v5.chriskrycho.com/journal/making-illegal-states-unrepresentable-in-ts/))
- **Fail loud, fail fast, fail closed.** Invariant violations should throw, log, and alert — never silently continue. On error, roll back entire transactions; never partially commit. ([OWASP A10:2025](https://owasp.org/Top10/2025/A10_2025-Mishandling_of_Exceptional_Conditions/))
- **Timeouts everywhere.** Every operation that waits on something external needs a deadline. Use `AbortController` (JS), `context.WithTimeout` (Go), `asyncio.wait_for` (Python).
- **Single-flight for exclusive resources.** If only one operation should run at a time, enforce it structurally — not with "please don't call this twice" comments.
- **Assume external data is hostile.** Sensors lie, APIs return garbage, configs get typos. Parse at the boundary; after that, the type system carries the proof.
- **Classify errors, respond proportionally.** Programmer errors → crash. Operational errors → retry/degrade. Business rule violations → return to caller. Security violations → reject + audit. Never leak stack traces, SQL, or credentials in error responses.

## Language-Specific Guidance

When the user specifies a language, adapt patterns accordingly:

**TypeScript**: branded types (`type UserId = number & { __brand: "UserId" }`), discriminated unions, exhaustive switch with `assertNever`, `AbortController` for timeouts, `Zod`/`Valibot` for boundary parsing. See [reference.md § Branded types](reference.md) and [examples.md § Example 2](examples.md).

**Python**: `NewType`, `@dataclass(frozen=True)`, `match` statements (3.10+), `asyncio.wait_for` for timeouts, `Pydantic` for boundary parsing. See [examples.md § Example 5](examples.md) and [examples.md § Example 9](examples.md).

**Rust**: newtype pattern with `TryFrom`, `enum` for state machines, `Result`/`Option` for error handling, ownership for concurrency safety. See [reference.md § Illegal states (Rust)](reference.md) and [examples.md § Example 4](examples.md).

**Go**: unexported struct + exported constructor, `context.WithTimeout`, `sync.Mutex` / channels, `errgroup` for structured concurrency. See [examples.md § Example 10](examples.md) and [examples.md § Example 12](examples.md).

Provide idiomatic examples in the target language when illustrating patterns in the spec.

## Additional Resources

- For detailed pattern explanations with code, see [reference.md](reference.md)
- For complete copy-pasteable examples across languages, see [examples.md](examples.md)
