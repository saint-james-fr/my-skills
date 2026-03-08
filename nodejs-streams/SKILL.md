---
name: nodejs-streams
description: Reference guide for Node.js streams — Readable, Writable, Transform, Duplex, piping, backpressure, Web Streams interop, stream consumers, and advanced patterns. Use when working with streams, piping data, implementing custom streams, handling backpressure, processing large files, building data pipelines, composing Transform streams, or converting between Node.js and Web Streams.
---

# Node.js Streams Reference

Based on *Node.js Design Patterns* (4th Ed.), Chapter 6.

## Why Streams

| Benefit | Buffered API | Streams |
|---------|-------------|---------|
| Memory | Entire data in RAM | Chunk at a time |
| Time | Wait for full read → process → write | Process chunks as they arrive (assembly line) |
| Composability | Manual wiring | `.pipe()` / `pipeline()` — Lego bricks |

Streams are everywhere: `fs`, `http`, `zlib`, `crypto`, `net`, `process.stdin/stdout`.

## The Four Stream Types

| Type | Role | Key method to implement |
|------|------|------------------------|
| **Readable** | Data source | `_read(size)` |
| **Writable** | Data destination | `_write(chunk, enc, cb)` |
| **Duplex** | Both source and destination | `_read()` + `_write()` |
| **Transform** | Data transformation (Duplex) | `_transform(chunk, enc, cb)` + `_flush(cb)` |

All streams are EventEmitters. Two operating modes: **binary** (Buffer/string) and **object** (`objectMode: true`).

## Reading from a Readable

### Non-flowing (paused) mode — pull

```typescript
process.stdin
  .on('readable', () => {
    let chunk
    while ((chunk = process.stdin.read()) !== null) {
      console.log(`Chunk: ${chunk.toString()}`)
    }
  })
  .on('end', () => console.log('done'))
```

### Flowing mode — push

```typescript
process.stdin
  .on('data', chunk => console.log(chunk.toString()))
  .on('end', () => console.log('done'))
```

### Async iterator (preferred for consuming entire stream)

```typescript
for await (const chunk of process.stdin) {
  console.log(chunk.toString())
}
```

## Implementing a Custom Readable

```typescript
import { Readable } from 'stream'

class RandomStream extends Readable {
  _read(size: number) {
    const chunk = crypto.randomBytes(size)
    this.push(chunk)
    if (Math.random() < 0.05) this.push(null) // signal EOF
  }
}
```

### Simplified construction

```typescript
const stream = new Readable({
  read(size) {
    this.push(crypto.randomBytes(size))
    if (Math.random() < 0.05) this.push(null)
  }
})
```

### From iterables

```typescript
import { Readable } from 'stream'
const stream = Readable.from([{ name: 'a' }, { name: 'b' }])
// objectMode: true by default with Readable.from()
```

## Writing to a Writable

```typescript
writable.write(chunk, encoding?, callback?)  // returns boolean
writable.end(chunk?, encoding?, callback?)   // signal no more data
```

### Handling Backpressure

When `write()` returns `false`, the internal buffer has exceeded `highWaterMark`. **Stop writing** and wait for the `drain` event.

```typescript
function generateData(writable: Writable) {
  while (hasMoreData()) {
    const shouldContinue = writable.write(nextChunk())
    if (!shouldContinue) {
      return writable.once('drain', () => generateData(writable))
    }
  }
  writable.end()
}
```

## Implementing a Custom Writable

```typescript
import { Writable } from 'stream'

const toFile = new Writable({
  objectMode: true,
  write(chunk: { path: string; content: string }, enc, cb) {
    mkdir(dirname(chunk.path), { recursive: true })
      .then(() => writeFile(chunk.path, chunk.content))
      .then(() => cb())
      .catch(cb)
  }
})
```

## Transform Streams

The workhorse for data pipelines. Receives data on Writable side, pushes transformed data on Readable side.

```typescript
import { Transform } from 'stream'

class ReplaceStream extends Transform {
  private tail = ''

  constructor(private searchStr: string, private replaceStr: string) {
    super()
  }

  _transform(chunk: Buffer, enc: string, cb: () => void) {
    const pieces = (this.tail + chunk.toString()).split(this.searchStr)
    const lastPiece = pieces[pieces.length - 1]
    const tailLen = this.searchStr.length - 1
    this.tail = lastPiece.slice(-tailLen)
    pieces[pieces.length - 1] = lastPiece.slice(0, -tailLen)
    this.push(pieces.join(this.replaceStr))
    cb()
  }

  _flush(cb: () => void) {
    this.push(this.tail)
    cb()
  }
}
```

### Simplified Transform

```typescript
const uppercasify = new Transform({
  transform(chunk, enc, cb) {
    this.push(chunk.toString().toUpperCase())
    cb()
  }
})
```

### Common Transform use cases
- **Filter**: push only chunks matching a condition
- **Aggregate**: accumulate in `_transform`, emit in `_flush`
- **Map**: transform each chunk 1:1

## PassThrough Streams

A Transform that passes data through unchanged. Useful for:
- **Observability**: attach listeners without altering data
- **Late piping**: provide a placeholder writable, pipe into it later
- **Lazy streams**: delay expensive resource init until data is consumed

```typescript
import { PassThrough } from 'stream'
const placeholder = new PassThrough()
upload('file.gz', placeholder) // starts upload, waits for data
createReadStream('file').pipe(createGzip()).pipe(placeholder)
```

## Piping

### `.pipe()`

```typescript
readable.pipe(transform).pipe(writable)
```

- Auto-handles backpressure
- Returns the destination stream (enables chaining)
- Does **not** propagate errors — must handle per-stream

### `pipeline()` (preferred)

```typescript
import { pipeline } from 'stream'

pipeline(
  createReadStream('input.txt'),
  createGzip(),
  createWriteStream('input.txt.gz'),
  (err) => {
    if (err) console.error('Pipeline failed:', err)
    else console.log('Pipeline succeeded')
  }
)
```

- Propagates errors and **destroys all streams** on failure
- Can be promisified: `const pipelineP = promisify(pipeline)`

### Error Handling: `.pipe()` vs `pipeline()`

| `.pipe()` | `pipeline()` |
|-----------|-------------|
| Must attach `error` listener to **each** stream | Single callback for all errors |
| Failing stream is unpiped but not destroyed | All streams destroyed on error |
| Can leak file descriptors / memory | Proper cleanup guaranteed |

## Advanced Piping Patterns

### Forking (one source → multiple destinations)

```typescript
const readable = createReadStream('file.txt')
readable.pipe(createWriteStream('copy1.txt'))
readable.pipe(createWriteStream('copy2.txt'))
readable.pipe(createGzip()).pipe(createWriteStream('file.txt.gz'))
```

### Merging (multiple sources → one destination)

```typescript
const dest = createWriteStream('merged.txt')
for (const file of files) {
  const src = createReadStream(file)
  src.pipe(dest, { end: false })
  await new Promise(resolve => src.on('end', resolve))
}
dest.end()
```

### Multiplexing / Demultiplexing

Combine multiple streams over a single channel (e.g., TCP), using framing (length-prefixed chunks with channel ID).

## Async Control Flow with Streams

### Sequential execution

`_transform()` processes one chunk at a time. The next chunk waits for `cb()`.

```typescript
Readable.from(files)
  .pipe(new Transform({
    objectMode: true,
    transform(filename, enc, done) {
      const src = createReadStream(filename)
      src.pipe(dest, { end: false })
      src.on('end', done)
    }
  }))
```

### Parallel execution with streams

Call `done()` immediately in `_transform()` to allow the next chunk to begin processing before the current one finishes. Track completion with a counter.

### Limited parallel execution with streams

Same as parallel, but gate `done()` with a concurrency counter. When at limit, defer `done()` until a slot opens.

## Quick Reference: Events

| Stream Type | Key Events |
|-------------|------------|
| Readable | `readable`, `data`, `end`, `error`, `close` |
| Writable | `drain`, `finish`, `error`, `close`, `pipe`, `unpipe` |
| Transform | All of the above (both sides) |

## Web Streams (WHATWG Streams Standard)

The **WHATWG Streams Standard** provides `ReadableStream`, `WritableStream`, and `TransformStream` — a universal streaming API for browsers and Node.js.

| Node.js Stream | Web Stream | Import from |
|---|---|---|
| `Readable` | `ReadableStream` | `node:stream/web` |
| `Writable` | `WritableStream` | `node:stream/web` |
| `Transform` | `TransformStream` | `node:stream/web` |

**When to use which**:
- **Browser code or `fetch` responses** → Web Streams (native to the platform)
- **Node.js server code** → Node.js streams (deeper ecosystem adoption)
- **Cross-platform libraries** → convert between them as needed

### Converting Node.js Streams → Web Streams

```typescript
import { Readable, Writable, Transform } from 'node:stream'

const webReadable = Readable.toWeb(nodeReadable)     // ReadableStream
const webWritable = Writable.toWeb(nodeWritable)     // WritableStream
const webTransform = Transform.toWeb(nodeTransform)  // TransformStream
```

### Converting Web Streams → Node.js Streams

```typescript
import { Readable, Writable, Transform } from 'node:stream'
import { ReadableStream, WritableStream, TransformStream } from 'node:stream/web'

const nodeReadable = Readable.fromWeb(webReadable)     // Readable
const nodeWritable = Writable.fromWeb(webWritable)     // Writable
const nodeTransform = Transform.fromWeb(webTransform)  // Transform
```

Conversions **wrap** the source (Adapter pattern) — they don't destroy it. Both the source and the converted stream emit the same chunks.

## Stream Consumer Utilities (`node:stream/consumers`)

When you need the **entire** stream content in memory (e.g., JSON parsing an HTTP response), use the built-in consumers instead of manual chunk accumulation.

```typescript
import consumers from 'node:stream/consumers'
```

| Method | Accumulates as |
|---|---|
| `consumers.json(stream)` | Parsed JSON object |
| `consumers.text(stream)` | String |
| `consumers.buffer(stream)` | Buffer |
| `consumers.arrayBuffer(stream)` | ArrayBuffer |
| `consumers.blob(stream)` | Blob |

Each returns a Promise that resolves when the stream is fully consumed. Works with both Node.js `Readable` and Web `ReadableStream`.

```typescript
import { request } from 'node:http'
import consumers from 'node:stream/consumers'

const req = request('http://example.com/data.json', async (res) => {
  const data = await consumers.json(res)
  console.log(data)
})
req.end()
```

With `fetch`, the response object has built-in consumers (`.json()`, `.text()`, `.blob()`, `.arrayBuffer()`):

```typescript
const res = await fetch('http://example.com/data.json')
const data = await res.json()
```

## Decision Guide

| Task | Approach |
|------|----------|
| Read large file chunk by chunk | `createReadStream` |
| Process data line by line | `readline` or custom Transform |
| Compress/encrypt on the fly | `pipeline(source, transform, dest)` |
| Convert iterable to stream | `Readable.from(iterable)` |
| Placeholder for late data | `PassThrough` |
| Custom binary protocol | Duplex stream |
| Data transformation | Transform stream |
| HTTP request/response body | Already streams — pipe directly |
| Consume full stream as JSON/text/buffer | `node:stream/consumers` |
| Browser-compatible streaming | Web Streams (`ReadableStream`, etc.) |
| Convert between Node.js and Web Streams | `.toWeb()` / `.fromWeb()` |

## Detailed Reference

For more advanced examples (parallel streams, multiplexing, combined streams), see [streams-detail.md](references/streams-detail.md).
