# Advanced Recipes — Detailed Reference

## Pre-Initialization Queues with State Pattern

Split the component into two states to remove boilerplate from business logic:

```typescript
import { EventEmitter } from 'events'

const METHODS_REQUIRING_CONNECTION = ['query']
const deactivate = Symbol('deactivate')

class InitializedState {
  async query(queryString: string) {
    console.log(`Query executed: ${queryString}`)
  }
}

class QueuingState {
  private commandsQueue: Array<() => void> = []

  constructor(private db: DB) {
    METHODS_REQUIRING_CONNECTION.forEach(methodName => {
      (this as any)[methodName] = (...args: unknown[]) => {
        return new Promise((resolve, reject) => {
          this.commandsQueue.push(() => {
            (db as any)[methodName](...args).then(resolve, reject)
          })
        })
      }
    })
  }

  [deactivate]() {
    this.commandsQueue.forEach(cmd => cmd())
    this.commandsQueue = []
  }
}

class DB extends EventEmitter {
  private state: any

  constructor() {
    super()
    this.state = new QueuingState(this)
  }

  async query(queryString: string) {
    return this.state.query(queryString)
  }

  connect() {
    setTimeout(() => {
      this.emit('connected')
      const oldState = this.state
      this.state = new InitializedState()
      oldState[deactivate]?.()
    }, 500)
  }
}
```

The `QueuingState` dynamically creates queuing versions of all methods listed in `METHODS_REQUIRING_CONNECTION`. When the DB connects, state switches to `InitializedState` and the queue is flushed.

## ProcessPool (Full Implementation)

```typescript
import { fork, ChildProcess } from 'child_process'

type Waiter = { resolve: (w: ChildProcess) => void; reject: (e: Error) => void }

export class ProcessPool {
  private pool: ChildProcess[] = []
  private active: ChildProcess[] = []
  private waiting: Waiter[] = []

  constructor(private file: string, private poolMax: number) {}

  acquire(): Promise<ChildProcess> {
    return new Promise((resolve, reject) => {
      if (this.pool.length > 0) {
        const worker = this.pool.pop()!
        this.active.push(worker)
        return resolve(worker)
      }

      if (this.active.length >= this.poolMax) {
        return this.waiting.push({ resolve, reject })
      }

      const worker = fork(this.file)
      worker.once('message', message => {
        if (message === 'ready') {
          this.active.push(worker)
          return resolve(worker)
        }
        worker.kill()
        reject(new Error('Improper process start'))
      })
      worker.once('exit', () => {
        this.active = this.active.filter(w => w !== worker)
        this.pool = this.pool.filter(w => w !== worker)
      })
    })
  }

  release(worker: ChildProcess) {
    if (this.waiting.length > 0) {
      const { resolve } = this.waiting.shift()!
      return resolve(worker)
    }
    this.active = this.active.filter(w => w !== worker)
    this.pool.push(worker)
  }
}
```

### Key behaviors

- **Reuses processes**: Released workers go back to the pool instead of being killed.
- **Limits concurrency**: When `active.length >= poolMax`, requests are queued in `waiting`.
- **Ready handshake**: Worker must send `'ready'` message before being assigned work.
- **Auto-cleanup**: Exited workers are removed from both `pool` and `active` arrays.

### Production improvements to add:
- Terminate idle processes after inactivity timeout
- Kill and replace non-responsive workers
- Configurable startup timeout

## ThreadPool (Worker Threads Variant)

Identical structure to ProcessPool with two differences:

```typescript
import { Worker } from 'worker_threads'

// In acquire():
// - Replace fork(this.file) with new Worker(this.file)
// - Replace 'message' ready handshake with 'online' event:
worker.once('online', () => {
  this.active.push(worker)
  resolve(worker)
})
```

### child_process.fork vs Worker Threads

| API | `child_process.fork` | `worker_threads` |
|-----|---------------------|-----------------|
| Create | `fork(file)` | `new Worker(file)` |
| Send to child | `worker.send(data)` | `worker.postMessage(data)` |
| Receive in child | `process.on('message', ...)` | `parentPort.on('message', ...)` |
| Send from child | `process.send(data)` | `parentPort.postMessage(data)` |
| Ready signal | Manual `process.send('ready')` | `'online'` event (automatic) |
| Data transfer | Structured clone (serialized) | Structured clone + `ArrayBuffer` transfer + `SharedArrayBuffer` |

## AbortController Cancelation Patterns

### AbortController vs AbortSignal: Separation of Concerns

```
┌─────────────────────────────────┐    ┌──────────────────────────────────┐
│       AbortController           │    │         AbortSignal              │
│  (owned by the caller)          │    │  (passed to the async operation) │
│                                 │    │                                  │
│  ac.abort()       → triggers ───┼───►│  signal.aborted         (bool)  │
│  ac.abort(reason) → with error  │    │  signal.throwIfAborted() (throw)│
│                                 │    │  signal.addEventListener('abort')│
└─────────────────────────────────┘    └──────────────────────────────────┘
```

### Full Cancelable Operation Example

```typescript
async function fetchUserData(userId: string, signal: AbortSignal) {
  signal.throwIfAborted()
  const user = await fetchUser(userId)

  signal.throwIfAborted()
  const posts = await fetchPosts(user.id)

  signal.throwIfAborted()
  const comments = await fetchComments(posts[0].id)

  return { user, posts, comments }
}

const ac = new AbortController()
setTimeout(() => ac.abort(), 2000)

try {
  const data = await fetchUserData('123', ac.signal)
} catch (err) {
  if (err instanceof Error && err.name === 'AbortError') {
    console.log('Operation canceled')
  } else {
    throw err
  }
}
```

### Reusable `callIfNotAborted` Wrapper

```typescript
const callIfNotAborted = <T>(
  signal: AbortSignal,
  fn: (...args: unknown[]) => Promise<T>,
  ...args: unknown[]
): Promise<T> => {
  signal.throwIfAborted()
  return fn(...args)
}

async function fetchUserData(userId: string, signal: AbortSignal) {
  const user = await callIfNotAborted(signal, fetchUser, userId)
  const posts = await callIfNotAborted(signal, fetchPosts, user.id)
  const comments = await callIfNotAborted(signal, fetchComments, posts[0].id)
  return { user, posts, comments }
}
```

### AbortSignal.timeout() — Auto-Aborting After a Duration

Creates a signal that fires automatically. More reliable than a plain `timeout` option — aborts even when the server drip-feeds data at very slow rates.

```typescript
const response = await fetch(url, {
  method: 'HEAD',
  signal: AbortSignal.timeout(5000),
})
```

### Composing Signals with AbortSignal.any()

Combine multiple abort reasons (user cancel + timeout):

```typescript
const userAc = new AbortController()
cancelButton.onclick = () => userAc.abort()

const combinedSignal = AbortSignal.any([
  userAc.signal,
  AbortSignal.timeout(30_000),
])

await fetchUserData('123', combinedSignal)
```

### Event-Driven Cleanup with AbortSignal

Register teardown logic that runs when cancelation fires:

```typescript
async function monitorSensor(signal: AbortSignal) {
  const connection = await openSensorConnection()

  signal.addEventListener('abort', () => {
    connection.close()
  }, { once: true })

  while (!signal.aborted) {
    const reading = await connection.nextReading()
    processSensorData(reading)
  }
}
```

### Why AbortController Over Custom Solutions

| Concern | Custom `cancelObj` | AbortController |
|---|---|---|
| Interoperability | Only works within your code | Standard API — works with `fetch`, streams, `node:test`, etc. |
| Error semantics | Custom `CancelError` class | Standard `DOMException` with `name: 'AbortError'` |
| Timeout support | Manual `setTimeout` + flag | `AbortSignal.timeout(ms)` built-in |
| Composability | Manual | `AbortSignal.any()` combines multiple signals |
| Ecosystem adoption | None | `fetch`, `EventTarget`, `node:readline`, `node:stream`, etc. |

## Batching + Caching: Combined Pattern

The key insight is that caching the **Promise** (not the value) gives both batching and caching:

```
Request 1 (product=book): Cache miss → start query, store Promise
Request 2 (product=book): Cache HIT → return same Promise (batching!)
Request 3 (product=book, 10s later): Cache HIT → return settled Promise (caching!)
Request 4 (product=book, 60s later): Cache miss (TTL expired) → new query
```

### Why cache the Promise, not the value?

1. **Batching is free**: Before the Promise settles, concurrent lookups return the same in-flight Promise
2. **No Zalgo**: A settled Promise always invokes `.then()` asynchronously
3. **Error handling is natural**: Failed Promises can be removed from cache immediately

### Generic wrapper

```typescript
const createBatchingCache = <K, V>(
  fn: (key: K) => Promise<V>,
  ttl = 30_000
) => {
  const cache = new Map<K, Promise<V>>()

  return (key: K): Promise<V> => {
    if (cache.has(key)) return cache.get(key)!

    const promise = fn(key)
    cache.set(key, promise)
    promise.then(
      () => setTimeout(() => cache.delete(key), ttl),
      () => cache.delete(key)
    )
    return promise
  }
}

// Usage
const cachedTotalSales = createBatchingCache(totalSalesRaw, 30_000)
```
