# Streams — Detailed Reference

## Buffered vs Streaming: Gzip Example

### Buffered (breaks on large files)

```typescript
const data = await readFile(filename)
const gzipped = await gzipPromise(data)
await writeFile(`${filename}.gz`, gzipped)
```

Fails with `ERR_FS_FILE_TOO_LARGE` on files > ~2GB.

### Streaming (constant memory)

```typescript
createReadStream(filename)
  .pipe(createGzip())
  .pipe(createWriteStream(`${filename}.gz`))
  .on('finish', () => console.log('Done'))
```

Works with any file size.

## Composability: Adding Encryption

Just insert another Transform into the pipeline:

```typescript
// Client: read → gzip → encrypt → send
createReadStream(filename)
  .pipe(createGzip())
  .pipe(createCipheriv('aes192', secret, iv))
  .pipe(req)

// Server: receive → decrypt → gunzip → write
req
  .pipe(createDecipheriv('aes192', secret, iv))
  .pipe(createGunzip())
  .pipe(createWriteStream(destFilename))
```

## Filtering and Aggregating with Transform Streams

### CSV Processing Example

Given a large CSV with `type,country,profit`, calculate total profit for Italy:

```typescript
const csvParser = new Transform({
  objectMode: true,
  transform(chunk, enc, cb) {
    const lines = chunk.toString().split('\n')
    for (const line of lines) {
      const [type, country, profit] = line.split(',')
      if (country?.trim()) {
        this.push({ type: type.trim(), country: country.trim(), profit: parseFloat(profit) })
      }
    }
    cb()
  }
})

const filterByCountry = (country: string) => new Transform({
  objectMode: true,
  transform(record, enc, cb) {
    if (record.country === country) this.push(record)
    cb()
  }
})

const sumProfit = () => {
  let total = 0
  return new Transform({
    objectMode: true,
    transform(record, enc, cb) { total += record.profit; cb() },
    flush(cb) { this.push(total.toString()); cb() }
  })
}

pipeline(
  createReadStream('data.csv'),
  csvParser,
  filterByCountry('Italy'),
  sumProfit(),
  process.stdout,
  (err) => { if (err) console.error(err) }
)
```

## Implementing an Unordered Parallel Stream

Process chunks in parallel for higher throughput (when order doesn't matter):

```typescript
class ParallelStream extends Transform {
  private running = 0
  private terminateCb: (() => void) | null = null

  constructor(
    private userTransform: (chunk: unknown, enc: string, push: Function, done: Function) => void,
    opts?: TransformOptions
  ) {
    super({ objectMode: true, ...opts })
  }

  _transform(chunk: unknown, enc: string, done: () => void) {
    this.running++
    this.userTransform(chunk, enc, this.push.bind(this), this.onComplete.bind(this))
    done() // immediately allow next chunk
  }

  _flush(done: () => void) {
    if (this.running > 0) {
      this.terminateCb = done
    } else {
      done()
    }
  }

  private onComplete(err?: Error) {
    this.running--
    if (err) return this.emit('error', err)
    if (this.running === 0) this.terminateCb?.()
  }
}
```

### Usage: URL Status Checker

```typescript
const checkUrls = new ParallelStream(
  async (url, enc, push, done) => {
    try {
      const res = await fetch(url as string)
      push(`${url}: ${res.status}\n`)
    } catch (err) {
      push(`${url}: ERROR\n`)
    }
    done()
  }
)

pipeline(
  Readable.from(urls),
  checkUrls,
  createWriteStream('results.txt'),
  (err) => { if (err) console.error(err) }
)
```

## Ordered Parallel Execution

To maintain chunk order while processing in parallel, buffer completed chunks and emit them only when all preceding chunks are also complete. Libraries like `parallel-transform` implement this.

## Limited Parallel Execution with Streams

Gate concurrency inside `_transform`:

```typescript
_transform(chunk: unknown, enc: string, done: () => void) {
  this.running++
  this.userTransform(chunk, enc, this.push.bind(this), () => {
    this.running--
    // Resume if we were at the limit
    if (this.pendingDone) {
      const pd = this.pendingDone
      this.pendingDone = null
      pd()
    }
  })

  if (this.running < this.concurrency) {
    done() // allow next chunk
  } else {
    this.pendingDone = done // hold until a slot opens
  }
}
```

## Combining Streams

Create a single Duplex stream from a pipeline of streams — the combined stream exposes the write side of the first stream and the read side of the last.

```typescript
import { PassThrough, pipeline } from 'stream'

function combine(...streams: (Readable | Transform | Writable)[]) {
  const input = new PassThrough()
  const output = new PassThrough()
  pipeline(input, ...streams, output, (err) => {
    if (err) output.destroy(err)
  })
  // Return a Duplex-like object
  return Object.assign(input, { pipe: output.pipe.bind(output) })
}
```

Use case: package a multi-step pipeline as a single reusable stream.

## Multiplexing and Demultiplexing

### Protocol: Length-prefixed framing

```
[1 byte: channel ID][4 bytes: data length][N bytes: data]
```

### Multiplexer (multiple sources → single channel)

```typescript
const multiplex = (sources: Readable[], dest: Writable) => {
  sources.forEach((source, i) => {
    source.on('data', (chunk: Buffer) => {
      const header = Buffer.alloc(5)
      header.writeUInt8(i, 0)
      header.writeUInt32BE(chunk.length, 1)
      dest.write(Buffer.concat([header, chunk]))
    })
  })
}
```

### Demultiplexer (single channel → multiple destinations)

Parse the header, route chunk to the correct destination by channel ID.

## Lazy Streams

Delay expensive initialization until data is actually consumed:

```typescript
import lazystream from 'lazystream'

const lazy = new lazystream.Readable(() => {
  return createReadStream('/very/large/file.bin')
})
// File descriptor NOT opened until lazy is piped or read
```

Use case: creating many stream handles at once (e.g., archiving thousands of files) without exhausting file descriptors.

## `highWaterMark` Reference

| Stream Type | Default | Controls |
|-------------|---------|----------|
| Readable | 16 KB | Internal read buffer before pausing `_read()` |
| Writable | 16 KB | Internal write buffer before `write()` returns `false` |
| Object mode | 16 objects | Count-based instead of byte-based |

Tune `highWaterMark` for throughput vs memory trade-off.
