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

## Infinite Recursive Promise Chains

When building recursive async algorithms (e.g., processing a queue item that spawns more items), deeply nested `.then()` chains can accumulate memory because each Promise holds a reference to the previous one. Solutions:

1. Use `async/await` with iterative loops instead of recursive `.then()` chains
2. Use `setImmediate()` to break the chain and let the event loop unwind the stack
3. Use a queue-based approach (TaskQueue) instead of recursion
