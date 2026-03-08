---
name: nodejs-async-patterns
description: Reference guide for Node.js asynchronous control flow patterns — callbacks, promises, async/await, concurrency control, and error handling. Use when writing async code, debugging callback issues, implementing sequential/parallel/limited-concurrent execution, dealing with race conditions, or choosing between callbacks, promises, and async/await.
---

# Node.js Async Patterns Reference

Based on _Node.js Design Patterns_ (4th Ed.), Chapters 3-5.

## The Golden Rule: Never Mix Sync and Async

An API must be **consistently** synchronous or asynchronous. Mixing both "unleashes Zalgo" — listeners registered after a sync callback fire never get called.

```typescript
// BAD: sync when cached, async when not
const inconsistentRead = (filename: string, cb: Function) => {
  if (cache.has(filename))
    cb(cache.get(filename)); // sync!
  else
    readFile(filename, "utf8", (err, data) => {
      cache.set(filename, data);
      cb(data);
    });
};
```

### Fixes

| Approach                          | When                                    |
| --------------------------------- | --------------------------------------- |
| Make it fully sync (direct style) | Pure CPU work, init-time config loading |
| Defer with `process.nextTick()`   | Guarantee async even from cache         |
| Use Promises / async-await        | Modern code — inherently async          |

```typescript
// GOOD: always async via deferred execution
if (cache.has(filename)) process.nextTick(() => cb(cache.get(filename)));
```

### `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)`

| API                 | Runs                                 | Risk                        |
| ------------------- | ------------------------------------ | --------------------------- |
| `process.nextTick`  | Before any I/O (microtask)           | I/O starvation if recursive |
| `setImmediate`      | After I/O events                     | Safe from starvation        |
| `setTimeout(fn, 0)` | Next event loop cycle (timers phase) | Slower than `setImmediate`  |

## Callback Conventions

1. **Callback comes last**: `readFile(path, opts, cb)`
2. **Error comes first**: `cb(err, result)` — always check `err`
3. **Early return on error**: `if (err) return cb(err)` — prevents deeper nesting
4. **Never throw inside async callbacks** — the exception jumps to the event loop, not the caller's try/catch
5. **Never invoke callback from inside try block** — catches unrelated errors from the callback itself

```typescript
// Correct error propagation pattern
readFile(filename, "utf8", (err, data) => {
  if (err) return cb(err);
  let parsed;
  try {
    parsed = JSON.parse(data);
  } catch (err) {
    return cb(err);
  }
  cb(null, parsed);
});
```

## Control Flow Patterns

### Sequential Execution (Callbacks)

```typescript
// Known set of tasks
const task1 = (cb: Function) => asyncOp(() => task2(cb));
const task2 = (cb: Function) => asyncOp(() => task3(cb));
const task3 = (cb: Function) => asyncOp(() => cb());
task1(() => console.log("all done"));

// Dynamic iteration (Sequential Iterator pattern)
const iterate = (index: number) => {
  if (index === tasks.length) return finish();
  tasks[index](() => iterate(index + 1));
};
iterate(0);
```

### Sequential Execution (Promises)

```typescript
// Dynamic chain
let promise = Promise.resolve();
for (const link of links) {
  promise = promise.then(() => spider(link, nesting - 1));
}
return promise;
```

### Sequential Execution (Async/Await)

```typescript
for (const link of links) {
  await spider(link, nesting - 1);
}
```

### Parallel Execution (Callbacks)

```typescript
let completed = 0;
tasks.forEach((task) => {
  task(() => {
    if (++completed === tasks.length) finish();
  });
});
```

### Parallel Execution (Promises)

```typescript
const promises = links.map((link) => spider(link, nesting - 1));
return Promise.all(promises);
```

### Parallel Execution (Async/Await)

```typescript
await Promise.all(links.map((link) => spider(link, nesting - 1)));
```

### Limited Parallel Execution (TaskQueue)

Spawn up to `concurrency` tasks at once. When one finishes, start the next.

```typescript
class TaskQueue extends EventEmitter {
  private running = 0;
  private queue: (() => Promise<unknown>)[] = [];

  constructor(private concurrency: number) {
    super();
  }

  runTask(task: () => Promise<unknown>) {
    return new Promise((resolve, reject) => {
      this.queue.push(() => task().then(resolve, reject));
      process.nextTick(() => this.next());
    });
  }

  private next() {
    if (this.running === 0 && this.queue.length === 0) {
      return this.emit("empty");
    }
    while (this.running < this.concurrency && this.queue.length) {
      const task = this.queue.shift()!;
      task().finally(() => {
        this.running--;
        this.next();
      });
      this.running++;
    }
  }
}
```

**In the wild**: `p-limit`, `p-map`, `async.queue`

## Race Conditions in Node.js

Even single-threaded, race conditions occur because of the **delay between async call and its callback**. Two tasks can both check a condition, then both act on the stale result.

```typescript
// Fix: use a Set to track in-progress URLs
const spidering = new Set<string>();
const spider = (url: string) => {
  if (spidering.has(url)) return;
  spidering.add(url);
  // ... proceed with download
};
```

## Promises Reference

### Key Properties

- `then()` returns a new Promise synchronously
- Exceptions in `onFulfilled`/`onRejected` auto-reject the returned Promise
- Errors propagate down the chain until caught
- Callbacks guaranteed async and invoked at most once

### Static Methods

| Method                         | Behavior                                                               |
| ------------------------------ | ---------------------------------------------------------------------- |
| `Promise.all(iterable)`        | Fulfills when **all** fulfill; rejects on **first** rejection. Other pending promises continue executing — they are **not** automatically canceled. |
| `Promise.allSettled(iterable)` | Waits for all to settle; returns `{ status, value/reason }[]`          |
| `Promise.race(iterable)`       | Settles with the **first** to settle. Useful for latency-based server selection. |
| `Promise.any(iterable)`        | Fulfills with the **first** to fulfill; rejects only if **all** reject |
| `Promise.resolve(val)`         | Wraps value/thenable in a Promise                                      |
| `Promise.reject(err)`          | Creates a rejected Promise                                             |
| `Promise.withResolvers()`      | Returns `{ promise, resolve, reject }` — control the promise externally (ES2024) |

### `Promise.withResolvers()` (ES2024)

Returns an object with a new Promise and its `resolve`/`reject` functions, letting you control the outcome externally. Useful when integrating with callback-based or event-driven code.

```typescript
const { promise, resolve, reject } = Promise.withResolvers<string>()

someEmitter.on('data', (val) => resolve(val))
someEmitter.on('error', (err) => reject(err))

const result = await promise
```

Especially handy in tests to track when async work completes:

```typescript
const task1status = Promise.withResolvers<void>()
const task = async () => {
  await someWork()
  task1status.resolve()
}
queue.pushTask(task)
await task1status.promise
```

### Promisification

```typescript
import { promisify } from "util";
const readFileP = promisify(readFile);
```

Or use `fs.promises` for the fs module directly.

## Async/Await Reference

### Key Rules

- `async` functions always return a Promise
- `await` pauses execution, yields to event loop, resumes when Promise settles
- `try/catch` catches both sync throws and async rejections uniformly
- **Anti-pattern**: `return promise` without `await` — local catch block won't fire

```typescript
// BAD: local catch won't trigger
async function bad() {
  try {
    return delayError(1000);
  } catch (err) {
    // no await!
    /* never reached */
  }
}

// GOOD: await before return
async function good() {
  try {
    return await delayError(1000);
  } catch (err) {
    console.error(err);
  }
}
```

### Anti-pattern: `Array.forEach` for Serial Execution

```typescript
// BAD: all tasks fire in parallel, not serial!
urls.forEach(async (url) => {
  await download(url);
});

// GOOD: use a for...of loop
for (const url of urls) {
  await download(url);
}
```

### Parallel with Async/Await

```typescript
// Unlimited
await Promise.all(urls.map((url) => download(url)));

// Limited (using p-map or TaskQueue)
const queue = new TaskQueue(4);
await Promise.all(urls.map((url) => queue.runTask(() => download(url))));
```

## Infinite Recursive Promise Chains (Memory Leak)

A `return` inside a `.then()` that calls itself creates an ever-growing chain of unsettled promises — a memory leak per the Promises/A+ spec.

```typescript
// BAD: leaks memory — the returned promise never settles
function leakingLoop(): Promise<void> {
  return delay(1).then(() => {
    console.log(`Tick ${Date.now()}`)
    return leakingLoop() // chain grows forever
  })
}
```

### Fixes

**1. Drop the `return`** — breaks the chain but loses error propagation:

```typescript
function nonLeakingLoop() {
  delay(1).then(() => {
    console.log(`Tick ${Date.now()}`)
    nonLeakingLoop()
  })
}
```

**2. Wrap in a Promise constructor** — no chain, but errors still propagate:

```typescript
function nonLeakingLoopWithErrors() {
  return new Promise((_resolve, reject) => {
    ;(function internalLoop() {
      delay(1)
        .then(() => {
          console.log(`Tick ${Date.now()}`)
          internalLoop()
        })
        .catch(reject)
    })()
  })
}
```

**3. `async`/`await` with `while` loop** (recommended) — clean, errors propagate:

```typescript
async function nonLeakingLoopAsync() {
  while (true) {
    await delay(1)
    console.log(`Tick ${Date.now()}`)
  }
}
```

The `async`/`await` recursive form **still leaks**:

```typescript
// BAD: same leak as the promise chain version
async function leakingLoopAsync() {
  await delay(1)
  return leakingLoopAsync()
}
```

**Rule of thumb**: for infinite async loops, use `while (true)` with `await`, never recursive `return`.

## Decision Guide

| Situation                             | Recommended Approach                     |
| ------------------------------------- | ---------------------------------------- |
| Modern code, any control flow         | `async/await`                            |
| Parallel with limit                   | `p-limit` / `p-map` / custom `TaskQueue` |
| Unlimited parallel                    | `Promise.all()`                          |
| Legacy callback API                   | Wrap with `util.promisify()`             |
| Event-driven (multiple notifications) | `EventEmitter`                           |
| Processing pipeline                   | Streams (see `nodejs-streams` skill)     |

## Additional Reference

For detailed code examples and the web spider case study, see [async-patterns-detail.md](references/async-patterns-detail.md).
