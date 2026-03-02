---
name: nodejs-advanced-recipes
description: Production recipes for common Node.js async problems — async component initialization (pre-init queues), request batching and caching, cancelable async operations, and running CPU-bound tasks with setImmediate, child processes, or worker threads. Use when initializing async components (DB connections), caching/batching API calls, canceling long-running operations, or offloading heavy computation.
---

# Node.js Advanced Recipes

Based on *Node.js Design Patterns* (3rd Ed.), Chapter 11.

## Quick Decision Guide

| Problem | Recipe | Jump to |
|---------|--------|---------|
| Component needs async init before use (DB, queue) | Pre-initialization queues | [Async Init](#async-init) |
| Duplicate concurrent requests to same API | Request batching | [Batching](#batching) |
| Reducing load from repeated identical calls | Request caching | [Caching](#caching) |
| Stopping a long-running async operation | Cancelable async functions | [Cancelation](#cancelation) |
| CPU-heavy task blocking the event loop | Worker threads / child processes | [CPU-Bound](#cpu-bound) |

---

## Async Component Initialization {#async-init}

**Problem**: A component (DB driver, message queue client) requires async setup before its methods can be called. Callers might invoke methods before init completes.

### Approach 1: Local Initialization Check

Check init status before every call. Simple but repetitive.

```typescript
import { once } from 'events'

const updateLastAccess = async () => {
  if (!db.connected) await once(db, 'connected')
  await db.query(`INSERT (${Date.now()}) INTO "LastAccesses"`)
}
```

### Approach 2: Delayed Startup

Wait for init at application boot, then proceed. Simple but delays startup and doesn't handle re-initialization.

```typescript
async function initialize() {
  db.connect()
  await once(db, 'connected')
}
await initialize()
// now safe to use db
```

### Approach 3: Pre-Initialization Queues (Recommended)

Queue method calls received before init; replay them once ready. The caller is completely unaware of init status.

```typescript
class DB extends EventEmitter {
  private connected = false
  private commandsQueue: Array<() => void> = []

  async query(queryString: string) {
    if (!this.connected) {
      return new Promise((resolve, reject) => {
        this.commandsQueue.push(() => {
          this.query(queryString).then(resolve, reject)
        })
      })
    }
    console.log(`Query executed: ${queryString}`)
  }

  connect() {
    setTimeout(() => {
      this.connected = true
      this.emit('connected')
      this.commandsQueue.forEach(cmd => cmd())
      this.commandsQueue = []
    }, 500)
  }
}
```

**Advanced**: Combine with the State pattern — `QueuingState` queues calls, `InitializedState` runs business logic. The DB class delegates to the current state and switches on init.

**In the wild**: Mongoose (MongoDB ORM) and `pg` (PostgreSQL) both use pre-init queues so you can call queries before the connection is established.

---

## Request Batching {#batching}

**Problem**: Multiple callers invoke the same async API with identical arguments concurrently, causing redundant work.

**Solution**: If a request for the same key is already in flight, return the existing Promise instead of starting a new one.

```typescript
import { totalSales as totalSalesRaw } from './totalSales'

const runningRequests = new Map<string, Promise<number>>()

export const totalSales = (product: string): Promise<number> => {
  if (runningRequests.has(product)) {
    return runningRequests.get(product)!
  }

  const resultPromise = totalSalesRaw(product)
  runningRequests.set(product, resultPromise)
  resultPromise.finally(() => runningRequests.delete(product))

  return resultPromise
}
```

**Why it works**: A Promise can have multiple `.then()` listeners, so all concurrent callers share the same result. The map entry is cleaned up when the request settles.

**Best for**: High-load applications with slow APIs where identical requests arrive within the same time window.

---

## Request Caching {#caching}

**Problem**: Same as batching, but requests are spread over time — batching alone won't help.

**Solution**: Keep the Promise in the map even after it settles. Add a TTL for invalidation.

```typescript
const CACHE_TTL = 30_000
const cache = new Map<string, Promise<number>>()

export const totalSales = (product: string): Promise<number> => {
  if (cache.has(product)) return cache.get(product)!

  const resultPromise = totalSalesRaw(product)
  cache.set(product, resultPromise)

  resultPromise.then(
    () => setTimeout(() => cache.delete(product), CACHE_TTL),
    () => cache.delete(product) // don't cache errors
  )

  return resultPromise
}
```

**Key insight**: Caching the **Promise** (not the resolved value) gives you batching + caching in one. Concurrent requests during the first load are batched automatically; subsequent requests hit the cache.

### Production Considerations

| Concern | Solution |
|---------|----------|
| Memory growth | LRU eviction (`lru-cache` npm) |
| Multi-process / multi-server | Shared store (Redis, Memcached) |
| Stale data | Event-driven invalidation vs. TTL |
| Zalgo | Promises are inherently async — safe by default |

---

## Cancelable Async Operations {#cancelation}

**Problem**: A long-running async operation needs to be stopped (user navigated away, request timed out, result is no longer needed).

### Approach 1: Manual Cancel Check

Check a cancel flag after each `await`:

```typescript
async function cancelable(cancelObj: { cancelRequested: boolean }) {
  const resA = await asyncRoutine('A')
  if (cancelObj.cancelRequested) throw new CancelError()
  const resB = await asyncRoutine('B')
  if (cancelObj.cancelRequested) throw new CancelError()
  const resC = await asyncRoutine('C')
}
```

### Approach 2: Cancel Wrapper (Less Boilerplate)

Wrap each async call with a function that checks for cancelation:

```typescript
const createCancelWrapper = () => {
  let cancelRequested = false
  const cancel = () => { cancelRequested = true }
  const cancelWrapper = <T>(fn: (...args: unknown[]) => Promise<T>, ...args: unknown[]) => {
    if (cancelRequested) return Promise.reject(new CancelError())
    return fn(...args)
  }
  return { cancelWrapper, cancel }
}

// Usage
const { cancelWrapper, cancel } = createCancelWrapper()
async function myOperation() {
  const a = await cancelWrapper(asyncRoutine, 'A')
  const b = await cancelWrapper(asyncRoutine, 'B')
}
setTimeout(cancel, 100)
```

### Approach 3: Generator-Based (Transparent Cancelation)

Use a generator where `yield` replaces `await`. A supervisor checks for cancelation between each yield:

```typescript
const cancelable = createAsyncCancelable(function* () {
  const resA = yield asyncRoutine('A')
  const resB = yield asyncRoutine('B')
  const resC = yield asyncRoutine('C')
})

const { promise, cancel } = cancelable()
setTimeout(cancel, 100)
```

No cancelation logic visible in the business code at all.

**In production**: Use the `caf` package (Cancelable Async Flows) — implements this pattern with edge cases handled.

---

## Running CPU-Bound Tasks {#cpu-bound}

**Problem**: A synchronous, CPU-intensive computation blocks the event loop, making the server unresponsive to all other requests.

### Approach 1: `setImmediate` Interleaving

Break the algorithm into steps; yield to the event loop between steps.

```typescript
_combineInterleaved(set: number[], subset: number[]) {
  this.runningCombine++
  setImmediate(() => {
    this._combine(set, subset)
    if (--this.runningCombine === 0) this.emit('end')
  })
}
```

| Pro | Con |
|-----|-----|
| Simple, no IPC overhead | Adds overhead per step |
| No extra processes/threads | Still runs on main thread (slower) |
| Good for light/sporadic tasks | Unsuitable for heavy or time-critical tasks |

**Never use `process.nextTick()`** for interleaving — it runs before I/O and causes I/O starvation.

### Approach 2: Child Processes (`child_process.fork`)

Run the algorithm in a separate Node.js process. Full CPU isolation.

```typescript
// worker.js
import { SubsetSum } from '../subsetSum'
process.on('message', msg => {
  const ss = new SubsetSum(msg.sum, msg.set)
  ss.on('match', data => process.send({ event: 'match', data }))
  ss.on('end', data => process.send({ event: 'end', data }))
  ss.start()
})
process.send('ready')
```

Use a **ProcessPool** to reuse processes and limit concurrency.

### Approach 3: Worker Threads (Recommended)

Lighter than processes — same V8 instance, smaller memory footprint, faster startup.

```typescript
// worker.js
import { parentPort } from 'worker_threads'
import { SubsetSum } from '../subsetSum'

parentPort!.on('message', msg => {
  const ss = new SubsetSum(msg.sum, msg.set)
  ss.on('match', data => parentPort!.postMessage({ event: 'match', data }))
  ss.on('end', data => parentPort!.postMessage({ event: 'end', data }))
  ss.start()
})
```

Use a **ThreadPool** (identical logic to ProcessPool but uses `new Worker()` instead of `fork()`).

### Comparison

| | `setImmediate` | `child_process.fork` | Worker Threads |
|--|----------------|---------------------|----------------|
| Isolation | None (same thread) | Full (separate process) | Thread-level |
| Memory overhead | None | High (~30MB per process) | Low (~few MB) |
| Startup time | None | Slow | Fast |
| Communication | N/A | `send()`/`on('message')` | `postMessage()`/`on('message')` |
| Multi-core | No | Yes | Yes |
| Best for | Light, sporadic tasks | Heavy tasks, non-Node workers | Heavy tasks, production use |

**In production**: Use `piscina` or `workerpool` for battle-tested thread/process pooling with error handling, timeouts, and retries.

---

## Detailed Reference

For full implementations (ProcessPool, ThreadPool, generator-based cancelation supervisor), see [advanced-recipes-detail.md](references/advanced-recipes-detail.md).
