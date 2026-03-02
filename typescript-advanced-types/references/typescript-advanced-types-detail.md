# TypeScript Advanced Types — Detail Reference

## Overloaded Function Types

Two syntaxes — full signature (supports overloads) and shorthand:

```typescript
// Shorthand
type Log = (message: string, userId?: string) => void

// Full (equivalent)
type Log = {
  (message: string, userId?: string): void
}
```

Overloading requires full signatures. The implementation must manually combine all overloads:

```typescript
type Reserve = {
  (from: Date, to: Date, destination: string): Reservation
  (from: Date, destination: string): Reservation
}

const reserve: Reserve = (
  from: Date,
  toOrDestination: Date | string,
  destination?: string,
) => {
  if (toOrDestination instanceof Date && destination !== undefined) {
    // round trip
  } else if (typeof toOrDestination === 'string') {
    // one-way
  }
}
```

The combined signature is not visible to callers — only the declared overloads are.

### Overload Resolution Order

TypeScript resolves overloads top-to-bottom. Literal overloads are hoisted above non-literal ones. Place more specific overloads first:

```typescript
type CreateElement = {
  (tag: 'a'): HTMLAnchorElement
  (tag: 'canvas'): HTMLCanvasElement
  (tag: 'table'): HTMLTableElement
  (tag: string): HTMLElement            // catchall last
}
```

### Callable Objects

Full signatures can include properties:

```typescript
type WarnUser = {
  (warning: string): void
  wasCalled: boolean
}
```

## Variance Deep Dive

### Shape Variance Example

```typescript
type ExistingUser = { id: number; name: string }
type NewUser = { name: string }

const deleteUser = (user: { id?: number; name: string }) => {
  delete user.id
}

const existing: ExistingUser = { id: 123, name: 'Ima User' }
deleteUser(existing) // Allowed — { id: number } <: { id?: number }
// But now existing.id is deleted at runtime while TS still thinks it's number!
```

TypeScript allows this because destructive updates are rare. Objects are covariant in their property types as a practical trade-off.

### Function Variance Derivation

Given `Crow <: Bird <: Animal`:

```typescript
const clone = (f: (b: Bird) => Bird): void => {
  const parent = new Bird()
  const baby = f(parent)
  baby.chirp()
}

clone((b: Bird): Crow => /* ... */)       // OK — return Crow <: Bird (covariant)
clone((b: Bird): Animal => /* ... */)     // Error — Animal is not <: Bird
clone((a: Animal): Bird => /* ... */)     // OK — Animal >: Bird (contravariant param)
clone((c: Crow): Bird => /* ... */)       // Error — Crow is not >: Bird
```

## Widening Behavior Table

| Declaration | Example | Inferred Type |
|-------------|---------|---------------|
| `let` / `var` | `let a = 'x'` | `string` |
| `const` (primitive) | `const a = 'x'` | `'x'` |
| `const` (object) | `const a = { x: 3 }` | `{ x: number }` |
| `as const` | `const a = { x: 3 } as const` | `{ readonly x: 3 }` |
| Explicit annotation | `let a: 'x' = 'x'` | `'x'` |
| `null` / `undefined` init | `let a = null` | `any` (narrowed on scope exit) |

### Preventing Widening on Reassignment

```typescript
const a = 'x'       // 'x'
let b = a            // string (widened!)

const c: 'x' = 'x'  // 'x'
let d = c            // 'x' (annotation preserved)
```

## Mapped Type Modifiers

The `-` operator removes modifiers. The `+` operator adds them (implied by default):

```typescript
type Account = {
  id: number
  isEmployee: boolean
  notes: string[]
}

type OptionalAccount = { [K in keyof Account]?: Account[K] }
type NullableAccount = { [K in keyof Account]: Account[K] | null }
type ReadonlyAccount = { readonly [K in keyof Account]: Account[K] }
type WritableAccount = { -readonly [K in keyof ReadonlyAccount]: Account[K] }
type RequiredAccount = { [K in keyof OptionalAccount]-?: Account[K] }
```

## Typesafe Deep Getter (Overloaded)

```typescript
type Get = {
  <O extends object, K1 extends keyof O>(o: O, k1: K1): O[K1]
  <O extends object, K1 extends keyof O, K2 extends keyof O[K1]>(
    o: O, k1: K1, k2: K2
  ): O[K1][K2]
  <
    O extends object,
    K1 extends keyof O,
    K2 extends keyof O[K1],
    K3 extends keyof O[K1][K2]
  >(o: O, k1: K1, k2: K2, k3: K3): O[K1][K2][K3]
}

const get: Get = (object: any, ...keys: string[]) => {
  let result = object
  keys.forEach(k => (result = result[k]))
  return result
}

get(activityLog, 'events', 0, 'type') // 'Read' | 'Write'
get(activityLog, 'bad')               // Error: '"bad"' not in keys
```

## Distributive Conditional Types — Step-by-Step

```typescript
type Without<T, U> = T extends U ? never : T

// Without<boolean | number | string, boolean>
// Step 1: distribute
//   Without<boolean, boolean> | Without<number, boolean> | Without<string, boolean>
// Step 2: evaluate
//   never | number | string
// Step 3: simplify
//   number | string
```

Without the distributive property (no conditional), the result would be `never`.

## infer — Practical Patterns

### Extract Array Element Type

```typescript
type ElementType<T> = T extends (infer U)[] ? U : T
type A = ElementType<number[]>  // number
type B = ElementType<string>    // string (not an array, falls through)
```

### Extract Function's Second Argument

```typescript
type SecondArg<F> = F extends (a: any, b: infer B) => any ? B : never
type F = typeof Array['prototype']['slice']
type A = SecondArg<F>  // number | undefined
```

### Extract Promise Resolution Type

```typescript
type Awaited<T> = T extends Promise<infer U> ? U : T
type A = Awaited<Promise<string>>  // string
```

## Branded Types — Full Pattern

```typescript
type CompanyID = string & { readonly brand: unique symbol }
type OrderID = string & { readonly brand: unique symbol }
type UserID = string & { readonly brand: unique symbol }
type ID = CompanyID | OrderID | UserID

// Constructors (companion object pattern)
const CompanyID = (id: string) => id as CompanyID
const OrderID = (id: string) => id as OrderID
const UserID = (id: string) => id as UserID

const queryForUser = (id: UserID) => { /* ... */ }

const companyId = CompanyID('8a6076cf')
const userId = UserID('d21b1dbf')
queryForUser(userId)    // OK
queryForUser(companyId) // Error: CompanyID not assignable to UserID
```

The brand is a phantom type — `string & { readonly brand: unique symbol }` is impossible to construct naturally, forcing use of the constructor functions.

## Safely Extending the Prototype

Two steps: augment the interface, then implement:

```typescript
// 1. Augment (in module mode, wrap in declare global)
declare global {
  interface Array<T> {
    zip<U>(list: U[]): [T, U][]
  }
}

// 2. Implement
Array.prototype.zip = function <T, U>(this: T[], list: U[]): [T, U][] {
  return this.map((v, k) => tuple(v, list[k]))
}

// Usage
import './zip'
;[1, 2, 3].map(n => n * 2).zip(['a', 'b', 'c'])
// [[2,'a'], [4,'b'], [6,'c']]
```

Exclude the file from `tsconfig.json` so consumers must explicitly import it.

## Key tsconfig Flags for Type Safety

| Flag | Effect |
|------|--------|
| `strict` | Enables all strict checks (superset of the below) |
| `strictFunctionTypes` | Contravariant function params (safe) |
| `strictNullChecks` | `null` and `undefined` are distinct types |
| `noImplicitReturns` | All code paths must return a value |
| `noUncheckedIndexedAccess` | Array/object index returns `T \| undefined` |
