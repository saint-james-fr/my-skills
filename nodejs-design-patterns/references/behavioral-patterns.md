# Behavioral Design Patterns — Detailed Reference

## Strategy

### Core Idea

Extract the *variable* logic into interchangeable **strategy** objects. The **context** holds common logic and delegates the variable part to the current strategy.

### Multi-Format Config Manager

```typescript
import { promises as fs } from 'fs'
import objectPath from 'object-path'

type FormatStrategy = {
  deserialize: (data: string) => Record<string, unknown>
  serialize: (data: Record<string, unknown>) => string
}

class Config {
  private data: Record<string, unknown> = {}

  constructor(private formatStrategy: FormatStrategy) {}

  get(path: string)              { return objectPath.get(this.data, path) }
  set(path: string, value: unknown) { return objectPath.set(this.data, path, value) }

  async load(filePath: string) {
    this.data = this.formatStrategy.deserialize(await fs.readFile(filePath, 'utf-8'))
  }

  async save(filePath: string) {
    await fs.writeFile(filePath, this.formatStrategy.serialize(this.data))
  }
}
```

```typescript
import ini from 'ini'

const iniStrategy: FormatStrategy  = { deserialize: ini.parse, serialize: ini.stringify }
const jsonStrategy: FormatStrategy = { deserialize: JSON.parse, serialize: d => JSON.stringify(d, null, 2) }

const config = new Config(iniStrategy) // or jsonStrategy
```

### Strategy Selection Options

- Pass at construction time (above)
- Two families: separate deserialization and serialization strategies
- Dynamic selection via a map (e.g. `extensionMap[ext]`)
- In simplest form, strategy is just a function: `function context(strategyFn) { ... }`

---

## State

### Core Idea

A specialization of Strategy where the active strategy **changes during the object's lifetime** based on internal state transitions.

### Failsafe Socket Example

```typescript
class FailsafeSocket {
  private queue: unknown[] = []
  private currentState!: SocketState
  private states: Record<string, SocketState>

  constructor(private options: ConnectionOptions) {
    this.states = {
      offline: new OfflineState(this),
      online:  new OnlineState(this),
    }
    this.changeState('offline')
  }

  changeState(state: string) {
    this.currentState = this.states[state]
    this.currentState.activate()
  }

  send(data: unknown) {
    this.currentState.send(data)
  }
}
```

**OfflineState**: queues data, retries connection every second. On connect → transition to `online`.

**OnlineState**: writes data to socket, flushes queued data on activate. On error → transition to `offline`.

### State Transition Control

Can be initiated by:
- The context object
- Client code
- The state objects themselves (best flexibility)

---

## Template

### Core Idea

A base class defines the algorithm skeleton. **Template methods** are abstract steps that subclasses implement. Unlike Strategy, the variation is fixed at class definition time, not at runtime.

```typescript
class ConfigTemplate {
  protected data: Record<string, unknown> = {}

  async load(file: string) {
    this.data = this._deserialize(await readFile(file, 'utf-8'))
  }

  async save(file: string) {
    await writeFile(file, this._serialize(this.data))
  }

  get(path: string)                 { return objectPath.get(this.data, path) }
  set(path: string, value: unknown) { return objectPath.set(this.data, path, value) }

  protected _serialize(_data: unknown): string   { throw new Error('Must implement') }
  protected _deserialize(_data: string): Record<string, unknown> { throw new Error('Must implement') }
}

class JsonConfig extends ConfigTemplate {
  protected _deserialize(data: string) { return JSON.parse(data) }
  protected _serialize(data: unknown)  { return JSON.stringify(data, null, 2) }
}
```

### Strategy vs Template

| Strategy | Template |
|----------|----------|
| Composition (has-a) | Inheritance (is-a) |
| Swap at runtime | Fixed at class definition |
| Context + separate strategy objects | Base class + subclass overrides |

### In the Wild

Node.js streams: `_read()`, `_write()`, `_transform()`, `_flush()` are all template methods.

---

## Iterator

### Core Idea

A standard protocol for traversing elements sequentially. In JS, it's based on protocols, not inheritance.

### The Iterator Protocol

An **iterator** is any object with a `next()` method returning `{ value, done }`.

```typescript
const createAlphabetIterator = () => {
  let code = 65 // 'A'
  return {
    next() {
      if (code > 90) return { done: true as const, value: undefined }
      return { value: String.fromCharCode(code++), done: false as const }
    }
  }
}
```

### The Iterable Protocol

An **iterable** has `[Symbol.iterator]()` returning an iterator. Enables `for...of`, spread, destructuring.

```typescript
class Matrix {
  constructor(private data: number[][]) {}

  *[Symbol.iterator]() {
    for (const row of this.data)
      for (const cell of row)
        yield cell
  }
}

for (const n of new Matrix([[1,2],[3,4]])) console.log(n)
```

### Generators

Syntactic sugar for creating iterators. `function*` + `yield`.

```typescript
function* range(start: number, end: number) {
  for (let i = start; i <= end; i++) yield i
}
```

### Async Iterators

Use `Symbol.asyncIterator` and `for await...of`. Works with Node.js streams.

```typescript
import { createReadStream } from 'fs'

const stream = createReadStream('file.txt', { encoding: 'utf-8' })
for await (const chunk of stream) {
  console.log(chunk)
}
```

### Readable.from()

Convert any iterable (sync or async) into a Readable stream:

```typescript
import { Readable } from 'stream'
const readable = Readable.from(range(1, 100))
```

---

## Middleware

### Core Idea

A pipeline of functions, each receiving data, processing it, and passing results to the next. The Node.js incarnation of **Chain of Responsibility**.

### Key Principles

1. Register middleware with `use()` — appended to the pipeline
2. New data triggers sequential async execution of all registered middleware
3. Each middleware can stop further processing (by not calling next or throwing)
4. Inbound and outbound middleware often run in **reverse order** to each other

### Middleware Manager

```typescript
class ZmqMiddlewareManager {
  private inbound:  MiddlewareFn[] = []
  private outbound: MiddlewareFn[] = []

  constructor(private socket: ZmqSocket) {
    this.handleIncoming()
  }

  use(mw: { inbound?: MiddlewareFn; outbound?: MiddlewareFn }) {
    if (mw.inbound)  this.inbound.push(mw.inbound)
    if (mw.outbound) this.outbound.unshift(mw.outbound) // reverse order
  }

  async send(message: unknown) {
    const final = await this.execute(this.outbound, message)
    return this.socket.send(final)
  }

  private async handleIncoming() {
    for await (const [msg] of this.socket) {
      await this.execute(this.inbound, msg)
    }
  }

  private async execute(pipeline: MiddlewareFn[], initial: unknown) {
    let msg = initial
    for (const fn of pipeline) { msg = await fn(msg) }
    return msg
  }
}
```

### Example Middleware

```typescript
// JSON serialization/deserialization
const jsonMiddleware = () => ({
  inbound:  (msg: Buffer) => JSON.parse(msg.toString()),
  outbound: (msg: unknown) => Buffer.from(JSON.stringify(msg)),
})

// Compression
const zlibMiddleware = () => ({
  inbound:  (msg: Buffer) => inflateRawAsync(Buffer.from(msg)),
  outbound: (msg: Buffer) => deflateRawAsync(msg),
})

manager.use(zlibMiddleware())
manager.use(jsonMiddleware())
```

Processing order: inbound = `zlib → json`, outbound = `json → zlib` (reversed).

---

## Command

### Core Idea

Wrap an action as an object with `run()`. Optionally add `undo()` and `serialize()`. Separates *intent* from *execution*.

### Four Components

| Component | Role |
|-----------|------|
| **Command** | Object encapsulating the action + params |
| **Client** | Creates the command |
| **Invoker** | Controls execution (immediate, delayed, remote, undo) |
| **Target/Receiver** | The service that actually performs the work |

### Task Pattern (Simplest Form)

```typescript
const createTask = (target: Function, ...args: unknown[]) =>
  () => target(...args)

// equivalent to:
const task = target.bind(null, ...args)
```

### Full Command with Undo and Serialization

```typescript
const createPostStatusCmd = (service: StatusService, status: string) => {
  let postId: string | null = null
  return {
    run()       { postId = service.postUpdate(status) },
    undo()      { if (postId) { service.destroyUpdate(postId); postId = null } },
    serialize() { return { type: 'status', action: 'post', status } },
  }
}
```

### Invoker

```typescript
class Invoker {
  private history: Command[] = []

  run(cmd: Command) {
    this.history.push(cmd)
    cmd.run()
  }

  undo() {
    const cmd = this.history.pop()
    cmd?.undo()
  }

  delay(cmd: Command, ms: number) {
    setTimeout(() => this.run(cmd), ms)
  }

  async runRemotely(cmd: Command) {
    await fetch('http://localhost:3000/cmd', {
      method: 'POST',
      body: JSON.stringify(cmd.serialize()),
    })
  }
}
```

### Use Cases

- **Undo/redo**: Keep command history, call `undo()` to revert
- **Task queues**: Serialize commands, send to workers
- **RPC**: `serialize()` → send over network → reconstruct on the other side
- **Transactions**: Group commands, execute atomically
- **Operational transformation**: Transform command sets for collaborative editing
