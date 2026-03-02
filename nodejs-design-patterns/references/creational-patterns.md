# Creational Design Patterns — Detailed Reference

## Factory

### Core Idea

A function that wraps `new`, decoupling the consumer from the concrete implementation. The consumer never touches the class directly.

### Key Benefits

- **Runtime type selection** — return different classes based on input/config
- **Encapsulation via closures** — private state inaccessible from outside
- **Small surface area** — a function is harder to misuse than a class

### Encapsulation with Closures

```typescript
const createPerson = (name: string) => {
  const privateProps: Record<string, unknown> = {}

  const person = {
    setName(n: string) {
      if (!n) throw new Error('A person must have a name')
      privateProps.name = n
    },
    getName() {
      return privateProps.name as string
    },
  }

  person.setName(name)
  return person
}
```

`privateProps` is invisible to the consumer. Only the returned interface can read/write it.

### Environment-Based Factory

```typescript
class Profiler {
  label: string
  lastTime: [number, number] | null = null
  constructor(label: string) { this.label = label }
  start() { this.lastTime = process.hrtime() }
  end() {
    const diff = process.hrtime(this.lastTime!)
    console.log(`Timer "${this.label}" took ${diff[0]}s ${diff[1]}ns`)
  }
}

const noopProfiler = { start() {}, end() {} }

export const createProfiler = (label: string) =>
  process.env.NODE_ENV === 'production' ? noopProfiler : new Profiler(label)
```

In production, profiling is silently disabled — zero overhead, zero consumer changes.

### Other Encapsulation Options

- Private class fields (`#field`) — native, modern
- WeakMaps — pre-ES2022 alternative
- Symbols — semi-private
- Underscore convention (`_field`) — no real protection, just convention

---

## Builder

### Core Idea

Replace a long constructor with a fluent, step-by-step interface. Each setter method returns `this`. A `build()` method produces the final object.

### Rules of Thumb

1. Group related parameters in a single setter (e.g. `setAuthentication(user, pass)`)
2. Deduce/implicitly set params (e.g. `withMotors()` sets `hasMotor = true`)
3. Normalize/validate in setters so the consumer doesn't have to
4. Keep the target class constructor private or at least discourage direct use

### URL Builder Example

```typescript
class UrlBuilder {
  private protocol?: string
  private username?: string
  private password?: string
  private hostname?: string
  private port?: number
  private pathname?: string
  private search?: string
  private hash?: string

  setProtocol(p: string)                  { this.protocol = p; return this }
  setAuthentication(user: string, pass: string) { this.username = user; this.password = pass; return this }
  setHostname(h: string)                  { this.hostname = h; return this }
  setPort(p: number)                      { this.port = p; return this }
  setPathname(p: string)                  { this.pathname = p; return this }
  setSearch(s: string)                    { this.search = s; return this }
  setHash(h: string)                      { this.hash = h; return this }

  build() {
    return new Url(this.protocol, this.username, this.password,
      this.hostname, this.port, this.pathname, this.search, this.hash)
  }
}

const url = new UrlBuilder()
  .setProtocol('https')
  .setAuthentication('user', 'pass')
  .setHostname('example.com')
  .build()
```

### Builder for Function Invocation

Same pattern, but replace `build()` with `invoke()`. Useful for wrapping complex function calls like `http.request()`.

---

## Revealing Constructor

### Core Idea

The constructor takes an **executor** function. The executor receives private internals (the "revealed members") that are unavailable after construction.

```
const obj = new SomeClass(function executor(revealedMembers) { ... })
```

### Immutable Buffer Example

```typescript
const MODIFIER_NAMES = ['swap', 'write', 'fill']

class ImmutableBuffer {
  [key: string]: unknown

  constructor(size: number, executor: (modifiers: Record<string, Function>) => void) {
    const buffer = Buffer.alloc(size)
    const modifiers: Record<string, Function> = {}

    for (const prop in buffer) {
      if (typeof (buffer as any)[prop] !== 'function') continue
      if (MODIFIER_NAMES.some(m => prop.startsWith(m))) {
        modifiers[prop] = (buffer as any)[prop].bind(buffer)
      } else {
        this[prop] = (buffer as any)[prop].bind(buffer)
      }
    }

    executor(modifiers)
  }
}

const immutable = new ImmutableBuffer(6, ({ write }) => { write('Hello!') })
// immutable exposes read methods only — write is gone
```

### Canonical Example: Promise

```typescript
new Promise((resolve, reject) => {
  // resolve and reject are "revealed" — no other way to change Promise state
})
```

---

## Singleton

### Module-Cached Instance

```typescript
// db-instance.ts
import { Database } from './database'
export const dbInstance = new Database('my-app-db', { url: 'localhost:5432' })
```

Every `import { dbInstance }` returns the same object due to module caching.

### Caveat: Not a True Singleton

If two packages depend on different *major versions* of a shared module, npm/yarn installs separate copies under each package's `node_modules`. Each copy has its own module cache entry → two instances.

```
app/
└── node_modules
    ├── package-a/node_modules/mydb  (v1.0.0)
    └── package-b/node_modules/mydb  (v2.0.0)
```

### When You Need a Real Global Singleton

```typescript
global.dbInstance = new Database(...)
```

Avoid unless absolutely necessary. Prefer DI or keep packages stateless.

---

## Dependency Injection

### Without DI (Tight Coupling)

```typescript
// blog.ts
import { db } from './db'  // hardcoded dependency
export class Blog {
  getAllPosts() { return db.all('SELECT * FROM posts') }
}
```

### With DI (Loose Coupling)

```typescript
// blog.ts — db is injected
export class Blog {
  constructor(private db: Database) {}
  getAllPosts() { return this.db.all('SELECT * FROM posts') }
}
```

```typescript
// index.ts — the injector
import { Blog } from './blog'
import { createDb } from './db'

const db = createDb('data.sqlite')
const blog = new Blog(db)
```

### Injection Styles

| Style | How |
|-------|-----|
| **Constructor injection** | `new Blog(db)` |
| **Function injection** | `getAllPosts(db)` |
| **Property injection** | `blog.db = db` |

### DI Containers (IoC)

For large apps, manual wiring becomes unmanageable. Libraries like **awilix** or **inversify** automate resolution order and lifecycle management.

### Trade-off

DI makes dependencies explicit and testable, but harder to trace at coding time. Choose DI when testability and swappability matter; use singletons for simple, small apps.
