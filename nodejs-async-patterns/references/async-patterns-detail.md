# Async Patterns — Detailed Reference

## The EventEmitter

Node.js's implementation of the Observer pattern. Allows objects to emit named events that cause registered listeners to be called.

```typescript
import { EventEmitter } from 'events'

class FindRegex extends EventEmitter {
  constructor(private regex: RegExp) { super() }

  addFile(file: string) {
    readFile(file, 'utf8', (err, content) => {
      if (err) return this.emit('error', err)
      this.emit('fileread', file)
      const match = content.match(this.regex)
      if (match) this.emit('found', file, match)
    })
    return this
  }
}

const finder = new FindRegex(/hello/i)
finder
  .addFile('file1.txt')
  .addFile('file2.txt')
  .on('found', (file, match) => console.log(`Matched in ${file}`))
  .on('error', err => console.error(err))
```

### EventEmitter vs Callbacks

| Use Callbacks when | Use EventEmitter when |
|--------------------|-----------------------|
| Single result/error to propagate | Multiple event types (data, error, end) |
| Result should be passed to exactly one consumer | Multiple listeners may react |
| One-shot async operation | Ongoing notifications over time |

### Memory Leaks

- Each listener adds to `_events` — too many listeners can leak memory
- Default warning at 11 listeners; set with `emitter.setMaxListeners(n)`
- Use `once()` for one-shot listeners — auto-removed after first invocation

## Callback Discipline

### Early Return Principle

```typescript
// Instead of nested if/else
if (err) {
  return cb(err)
}
// Happy path continues at top indent level
```

### Named Functions Over Anonymous

```typescript
// BAD: callback hell
asyncOp1((err, r1) => {
  asyncOp2(r1, (err, r2) => {
    asyncOp3(r2, (err, r3) => { ... })
  })
})

// GOOD: named, modular functions
const step1 = (cb: Function) => asyncOp1((err, r1) => {
  if (err) return cb(err)
  step2(r1, cb)
})
const step2 = (r1: unknown, cb: Function) => asyncOp2(r1, (err, r2) => {
  if (err) return cb(err)
  step3(r2, cb)
})
```

## Promise Chains — Sequential Async

```typescript
function download(url: string, filename: string) {
  return superagent.get(url)
    .then(res => {
      content = res.text
      return mkdirp(dirname(filename))
    })
    .then(() => writeFile(filename, content))
    .then(() => content)
}
```

Key: each `then()` returns a Promise, enabling the chain. Errors propagate until a `catch()` intercepts them.

## Async/Await — Sequential

```typescript
async function download(url: string, filename: string) {
  const res = await superagent.get(url)
  await mkdirp(dirname(filename))
  await writeFile(filename, res.text)
  return res.text
}
```

Equivalent to the promise chain but reads like synchronous code.

## TaskQueue with Promises (Full Implementation)

```typescript
import { EventEmitter } from 'events'

export class TaskQueue extends EventEmitter {
  private running = 0
  private queue: (() => Promise<unknown>)[] = []

  constructor(private concurrency: number) {
    super()
  }

  runTask<T>(task: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      this.queue.push(() => task().then(resolve, reject))
      process.nextTick(() => this.next())
    })
  }

  private next() {
    if (this.running === 0 && this.queue.length === 0) {
      return this.emit('empty')
    }
    while (this.running < this.concurrency && this.queue.length) {
      const task = this.queue.shift()!
      task()
        .catch(err => this.emit('error', err))
        .finally(() => {
          this.running--
          this.next()
        })
      this.running++
    }
  }
}
```

### Usage

```typescript
const queue = new TaskQueue(2)
queue.on('error', console.error)
queue.on('empty', () => console.log('All done'))

for (const url of urls) {
  queue.runTask(() => downloadAndProcess(url))
}
```

## Production Libraries

| Library | Purpose |
|---------|---------|
| `p-limit` | Limit concurrency of async functions |
| `p-map` | Parallel map with concurrency limit |
| `p-queue` | Priority queue with concurrency control |
| `async` (npm) | Comprehensive callback-based control flow |

## Infinite Recursive Promise Chains (Memory Leak)

A `return` inside `.then()` that calls itself creates an ever-growing chain of unsettled Promises — a memory leak per the Promises/A+ specification. No compliant implementation is immune.

This affects any task without a predefined ending: live stream processing, IoT sensor monitoring, crypto market feeds, or any recursive functional pattern.

### The Leak

```typescript
function leakingLoop(): Promise<void> {
  return delay(1).then(() => {
    console.log(`Tick ${Date.now()}`)
    return leakingLoop() // returned Promise depends on the next, which depends on the next...
  })
}

// Launch 1M instances → memory grows until crash
for (let i = 0; i < 1e6; i++) {
  leakingLoop()
}
```

The Promise returned by `leakingLoop()` never resolves because its status depends on the next invocation, which depends on the next, ad infinitum.

### Fix 1: Drop the `return` (breaks the chain)

```typescript
function nonLeakingLoop() {
  delay(1).then(() => {
    console.log(`Tick ${Date.now()}`)
    nonLeakingLoop() // no return → no chain
  })
}
```

**Trade-off**: errors from deep recursion are lost — no link between promises. Mitigate with extra logging inside the function.

### Fix 2: Promise constructor wrapper (preserves error propagation)

```typescript
function nonLeakingLoopWithErrors() {
  return new Promise((_resolve, reject) => {
    ;(function internalLoop() {
      delay(1)
        .then(() => {
          console.log(`Tick ${Date.now()}`)
          internalLoop()
        })
        .catch(reject) // errors at any depth surface to the caller
    })()
  })
}
```

No chain between internal promises, but the outer Promise still rejects if any step fails.

### Fix 3: `async`/`await` with `while` loop (recommended)

```typescript
async function nonLeakingLoopAsync() {
  while (true) {
    await delay(1)
    console.log(`Tick ${Date.now()}`)
  }
}
```

Cleanest solution. Errors propagate naturally via `try/catch`.

**The `async`/`await` recursive form still leaks:**

```typescript
// BAD: same chain leak
async function leakingLoopAsync() {
  await delay(1)
  return leakingLoopAsync() // return creates the same infinite chain
}
```

### Rule of Thumb

For infinite async loops, use `while (true)` with `await`. Never `return` from a recursive async call.

### Further Reading

- Node.js issue: [nodejsdp.link/node-6673](https://nodejsdp.link/node-6673)
- Promises/A+ spec issue: [nodejsdp.link/promisesaplus-memleak](https://nodejsdp.link/promisesaplus-memleak)

## `Promise.withResolvers()` (ES2024)

Returns `{ promise, resolve, reject }` — gives external control over a Promise's outcome. Equivalent to:

```typescript
function withResolvers<T>() {
  let resolve: (value: T) => void
  let reject: (reason: unknown) => void
  const promise = new Promise<T>((res, rej) => {
    resolve = res
    reject = rej
  })
  return { promise, resolve: resolve!, reject: reject! }
}
```

### Use Case: Bridging Event-Driven Code

```typescript
const { promise, resolve } = Promise.withResolvers<Buffer>()

socket.on('data', (chunk) => resolve(chunk))
socket.on('error', (err) => reject(err))

const firstChunk = await promise
```

### Use Case: Tracking Async Task Completion in Tests

```typescript
import { test } from 'node:test'
import { strict as assert } from 'node:assert'

test('All tasks complete and empty fires', async () => {
  const queue = new TaskQueue(2)

  const task1status = Promise.withResolvers<void>()
  let task1Done = false
  const task2status = Promise.withResolvers<void>()
  let task2Done = false

  queue.pushTask(async () => {
    await setImmediate()
    task1Done = true
    task1status.resolve()
  })
  queue.pushTask(async () => {
    await setImmediate()
    task2Done = true
    task2status.resolve()
  })

  await Promise.allSettled([task1status.promise, task2status.promise])
  assert.ok(task1Done)
  assert.ok(task2Done)
})
```

`Promise.withResolvers()` is ideal in tests when you need to observe or control when specific async operations complete, without wrapping everything in callbacks.
