---
name: typescript-advanced-types
description: Advanced TypeScript type system patterns including generics, bounded polymorphism, variance, conditional types, mapped types, branded types, type guards, and type-driven development. Use when writing complex generic utilities, designing type-safe APIs, debugging assignability errors, or working with mapped/conditional types.
---

# TypeScript Advanced Types

## Generics

### Call Signatures & Generic Placement

```typescript
// Shorthand — T bound per call
type Filter = <T>(array: T[], f: (item: T) => boolean) => T[]

// Full — T bound per call
type Filter = {
  <T>(array: T[], f: (item: T) => boolean): T[]
}

// T bound at alias level — must be supplied when referencing the type
type Filter<T> = (array: T[], f: (item: T) => boolean) => T[]
let numFilter: Filter<number> = (array, f) => // ...

// Named function — T bound per call
const filter = <T>(array: T[], f: (item: T) => boolean): T[] => { /* ... */ }
```

**Binding rules:** generics declared in a call signature are bound when the function is *called*; generics declared on the type alias are bound when the type is *used*.

### Multiple Generics

Use separate type params when input and output types differ:

```typescript
const map = <T, U>(array: T[], f: (item: T) => U): U[] => {
  const result: U[] = []
  for (let i = 0; i < array.length; i++) result[i] = f(array[i])
  return result
}
```

### Bounded Polymorphism (`extends`)

Constrain a generic to at least some shape:

```typescript
type TreeNode = { value: string }
type LeafNode = TreeNode & { isLeaf: true }
type InnerNode = TreeNode & { children: [TreeNode] | [TreeNode, TreeNode] }

const mapNode = <T extends TreeNode>(node: T, f: (v: string) => string): T => ({
  ...node,
  value: f(node.value),
})

mapNode({ value: 'a', isLeaf: true } as LeafNode, v => v.toUpperCase()) // LeafNode
```

**Multiple constraints** — intersect them:

```typescript
const logPerimeter = <S extends HasSides & SidesHaveLength>(s: S): S => {
  console.log(s.numberOfSides * s.sideLength)
  return s
}
```

**Modeling arity** with bounded rest params:

```typescript
const call = <T extends unknown[], R>(f: (...args: T) => R, ...args: T): R => f(...args)
```

### Generic Defaults

```typescript
type MyEvent<T extends HTMLElement = HTMLElement> = {
  target: T
  type: string
}
let ev: MyEvent = { target: el, type: 'click' } // T defaults to HTMLElement
```

Generics with defaults must come after those without (like optional params).

### Generic Inference Tips

- TypeScript infers generics from call-site arguments. If it can't, it falls back to the bound/default or `unknown`.
- When inference fails (e.g. `new Promise(resolve => resolve(45))`), annotate explicitly: `new Promise<number>(...)`.
- Explicit annotations are all-or-nothing: supply every generic or none.

## Variance

Four kinds: **invariance** (exactly T), **covariance** (<: T), **contravariance** (>: T), **bivariance** (<: T or >: T).

TypeScript rules:
- Objects, classes, arrays, return types → **covariant** in their members
- Function parameters and `this` → **contravariant** (with `strictFunctionTypes`)

A function A is a subtype of B when:
1. A's `this` is `>:` B's `this`
2. Each of A's params is `>:` its counterpart in B
3. A's return type is `<:` B's return type

```typescript
// Covariant return: Crow <: Bird, so (b: Bird) => Crow <: (b: Bird) => Bird ✓
// Contravariant param: Animal >: Bird, so (a: Animal) => Bird <: (b: Bird) => Bird ✓
// WRONG: Crow <: Bird, so (c: Crow) => Bird is NOT <: (b: Bird) => Bird ✗
```

Enable `strictFunctionTypes` (included in `strict`) for safe contravariant param checking.

## Type Widening & Narrowing

### Widening

```typescript
let a = 'x'          // string (widened)
const b = 'x'        // 'x' (literal, not widened)
const c = { x: 3 }   // { x: number } (properties still widen)
```

**`as const`** — opt out of widening recursively, marks everything `readonly`:

```typescript
const d = { x: 3 } as const      // { readonly x: 3 }
const e = [1, { x: 2 }] as const // readonly [1, { readonly x: 2 }]
```

### Excess Property Checking

Fresh object literals trigger excess property checks — catches typos in option bags:

```typescript
type Options = { baseURL: string; tier?: 'prod' | 'dev' }
new API({ baseURL: '...', tierr: 'prod' }) // Error: 'tierr' not in Options
```

Assigning to a variable first or using `as Options` bypasses this check.

### Refinement (Narrowing)

TypeScript uses control flow (`if`, `switch`, `typeof`, `instanceof`, `in`) to narrow types:

```typescript
const parseWidth = (width: number | string | null | undefined): Width | null => {
  if (width == null) return null            // narrows to number | string
  if (typeof width === 'number') return { unit: 'px', value: width }
  // width is now string
  const unit = parseUnit(width)
  if (unit) return { unit, value: parseFloat(width) }
  return null
}
```

### Discriminated Unions

Tag each member with a unique literal field for reliable narrowing:

```typescript
type TextEvent = { type: 'text'; value: string; target: HTMLInputElement }
type MouseEvent = { type: 'mouse'; value: [number, number]; target: HTMLElement }
type AppEvent = TextEvent | MouseEvent

const handle = (event: AppEvent) => {
  if (event.type === 'text') {
    event.value   // string
    event.target  // HTMLInputElement — fully narrowed
  }
}
```

## Totality (Exhaustiveness)

TypeScript catches missing cases. Use `noImplicitReturns` for stricter checking.

Exhaustive switch pattern:

```typescript
const assertNever = (x: never): never => { throw new Error(`Unexpected: ${x}`) }

const handle = (event: AppEvent) => {
  switch (event.type) {
    case 'text': return // ...
    case 'mouse': return // ...
    default: return assertNever(event) // compile error if a case is missing
  }
}
```

## Type Operators for Objects

### Keying In (`T[K]`)

```typescript
type FriendList = APIResponse['user']['friendList']
type Friend = FriendList['friends'][number]   // element type of an array
```

### `keyof`

```typescript
type UserKeys = keyof APIResponse['user']  // 'userId' | 'friendList'
```

### Typesafe Getter

```typescript
const get = <O extends object, K extends keyof O>(o: O, k: K): O[K] => o[k]
```

### Record

```typescript
type Weekday = 'Mon' | 'Tue' | 'Wed' | 'Thu' | 'Fri'
type Day = Weekday | 'Sat' | 'Sun'
const nextDay: Record<Weekday, Day> = { Mon: 'Tue', Tue: 'Wed', /* ... */ }
```

### Mapped Types

```typescript
type Optional<T> = { [K in keyof T]?: T[K] }
type Nullable<T> = { [K in keyof T]: T[K] | null }
type ReadonlyT<T> = { readonly [K in keyof T]: T[K] }
type Writable<T> = { -readonly [K in keyof T]: T[K] }  // minus removes readonly
type Required<T> = { [K in keyof T]-?: T[K] }           // minus removes optional
```

Built-in: `Record`, `Partial`, `Required`, `Readonly`, `Pick`.

## Conditional Types

```typescript
type IsString<T> = T extends string ? true : false
```

### Distributive Conditionals

When T is a union, the condition distributes over each member:

```typescript
type ToArray<T> = T extends unknown ? T[] : T[]
type A = ToArray<number | string>  // number[] | string[]

type Without<T, U> = T extends U ? never : T
type B = Without<boolean | number | string, boolean>  // number | string
```

### `infer` Keyword

Declare a type variable inline within a condition:

```typescript
type ElementType<T> = T extends (infer U)[] ? U : T
type A = ElementType<number[]>  // number

type SecondArg<F> = F extends (a: any, b: infer B) => any ? B : never
```

### Built-in Conditional Types

| Type | Purpose |
|------|---------|
| `Exclude<T, U>` | Types in T not assignable to U |
| `Extract<T, U>` | Types in T assignable to U |
| `NonNullable<T>` | Removes `null` and `undefined` |
| `ReturnType<F>` | Function return type |
| `InstanceType<C>` | Class instance type |
| `Parameters<F>` | Function parameter types as tuple |

## User-Defined Type Guards

Refinement doesn't survive scope boundaries. Use `is` to carry it:

```typescript
const isString = (a: unknown): a is string => typeof a === 'string'

const parseInput = (input: string | number) => {
  if (isString(input)) {
    input.toUpperCase()  // works — input narrowed to string
  }
}
```

## Branded / Nominal Types

TypeScript is structural. Simulate nominal types with type branding:

```typescript
type UserID = string & { readonly brand: unique symbol }
type OrderID = string & { readonly brand: unique symbol }

const UserID = (id: string) => id as UserID
const OrderID = (id: string) => id as OrderID

const queryForUser = (id: UserID) => { /* ... */ }
queryForUser(UserID('abc'))    // OK
queryForUser(OrderID('abc'))   // Error — OrderID is not assignable to UserID
```

Zero runtime overhead — brands exist only at compile time.

## Companion Object Pattern

Pair a type and a value with the same name (types and values live in separate namespaces):

```typescript
type Currency = { unit: 'EUR' | 'GBP' | 'JPY' | 'USD'; value: number }
const Currency = {
  DEFAULT: 'USD' as const,
  from(value: number, unit = Currency.DEFAULT): Currency {
    return { unit, value }
  },
}
// import { Currency } from './Currency' — imports both type and value
```

## Escape Hatches

Use sparingly:

| Mechanism | Syntax | Use case |
|-----------|--------|----------|
| Type assertion | `x as T` | Override inferred type (A <: B <: C only) |
| Nonnull assertion | `x!` | Strip `null \| undefined` when you're certain |
| Definite assignment | `let x!: T` | Tell TS a variable will be assigned before use |

Prefer refactoring over assertions: split types into discriminated unions instead of sprinkling `!`.

## Type-Driven Development

Sketch type signatures first, fill in implementations later:

```typescript
// 1. Define the shape
const map = <T, U>(array: T[], f: (item: T) => U): U[] => { /* TODO */ }
// 2. The signature already tells you what map does
// 3. Implementation follows naturally
```

## Tuple Inference Helper

Force TypeScript to infer a tuple instead of an array:

```typescript
const tuple = <T extends unknown[]>(...ts: T): T => ts
const a = tuple(1, true)  // [number, boolean], not (number | boolean)[]
```

For detailed examples, see [typescript-advanced-types-detail.md](references/typescript-advanced-types-detail.md).
