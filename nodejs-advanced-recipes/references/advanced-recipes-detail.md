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

## Generator-Based Cancelation Supervisor

The `createAsyncCancelable` function wraps a generator to add transparent cancelation:

```typescript
class CancelError extends Error {
  constructor() { super('Canceled') }
}

const createAsyncCancelable = <T>(generatorFn: (...args: unknown[]) => Generator) => {
  return (...args: unknown[]) => {
    const gen = generatorFn(...args)
    let cancelRequested = false

    const cancel = () => { cancelRequested = true }

    const promise = new Promise<T>((resolve, reject) => {
      async function nextStep(prevResult: IteratorResult<unknown>) {
        if (cancelRequested) return reject(new CancelError())
        if (prevResult.done) return resolve(prevResult.value as T)

        try {
          nextStep(gen.next(await prevResult.value))
        } catch (err) {
          try {
            nextStep(gen.throw(err))
          } catch (err2) {
            reject(err2)
          }
        }
      }
      nextStep({ value: undefined, done: false } as IteratorResult<unknown>)
    })

    return { promise, cancel }
  }
}
```

### How it works

1. The generator function uses `yield` instead of `await`
2. The supervisor iterates the generator, awaiting each yielded Promise
3. Between each yield, it checks `cancelRequested`
4. If canceled, it rejects with `CancelError` — no further yields are processed
5. Errors from rejected Promises are thrown into the generator via `gen.throw()`

### Usage

```typescript
const fetchData = createAsyncCancelable(function* () {
  const user = yield fetchUser(123)
  const posts = yield fetchPosts(user.id)
  const comments = yield fetchComments(posts[0].id)
  return { user, posts, comments }
})

const { promise, cancel } = fetchData()

promise.catch(err => {
  if (err instanceof CancelError) console.log('Operation canceled')
})

// Cancel after 2 seconds
setTimeout(cancel, 2000)
```

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
