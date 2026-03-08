---
name: nodejs-networking-io
description: Node.js low-level I/O covering Buffers, binary data, encodings, file system operations, TCP/UDP networking, HTTP internals, DNS, and TLS encryption. Use when working with binary protocols, raw sockets, file descriptors, file locking, network servers, or TLS certificates. Based on Node.js Design Patterns (4th Ed.).
---

# Node.js Networking & I/O

## Buffers

Buffers are raw heap allocations exposed as array-like objects. Globally available, no `require` needed.

### Creating & Converting

```typescript
// From string (default UTF-8)
const buf = Buffer.from('hello')

// With explicit encoding
const b64 = Buffer.from('am9obm55OmMtYmFk', 'base64')

// Allocate zeroed
const zeroed = Buffer.alloc(256)

// Allocate uninitialized (faster, use when you'll overwrite immediately)
const unsafe = Buffer.allocUnsafe(256)
```

### Supported Encodings

`utf8` (default), `ascii`, `base64`, `hex`, `utf16le`, `latin1`, `binary`.

### toString with Encoding & Range

```typescript
buf.toString('ascii')               // entire buffer as ASCII
buf.toString('utf8', 0, 11)         // bytes 0-10 as UTF-8
buf.toString('base64')              // entire buffer as base64
```

### Practical: Base64 Authentication Header

```typescript
const encoded = Buffer.from(`${user}:${pass}`).toString('base64')
// → 'am9obm55OmMtYmFk'
```

### Practical: Data URIs

```typescript
// Encode binary file to data URI
const data = fs.readFileSync('./image.png').toString('base64')
const uri = `data:image/png;base64,${data}`

// Decode data URI back to file
const raw = uri.split(',')[1]
fs.writeFileSync('./output.png', Buffer.from(raw, 'base64'))
```

### Reading Binary Data

Buffer indices are byte positions in memory:

```typescript
buf[0]                    // unsigned 8-bit integer at position 0
buf.readUInt8(offset)     // same, explicit method
buf.readUInt16LE(offset)  // unsigned 16-bit little-endian
buf.readUInt32LE(offset)  // unsigned 32-bit little-endian
buf.readInt16BE(offset)   // signed 16-bit big-endian
buf.readFloatLE(offset)   // 32-bit float little-endian
buf.readDoubleLE(offset)  // 64-bit double little-endian
```

### Subarray (View)

`buf.subarray(start, end)` returns a view — **not a copy**. Mutations affect the original.

```typescript
const fieldBuf = buf.subarray(32, 64) // bytes 32-63, shares memory with buf
```

> `buf.slice()` is deprecated (DEP0158) — use `buf.subarray()` instead. Same semantics.

### Bitmasks

```typescript
const bitmasks = [1, 2, 4, 8, 16, 32, 64, 128]
const byte = buf[0]
for (let i = 0; i < bitmasks.length; i++) {
  if ((byte & bitmasks[i]) === bitmasks[i]) {
    // bit i+1 is set
  }
}
```

### Compression with zlib

```typescript
import { deflate, inflate } from 'node:zlib'

// Compress
deflate(Buffer.from('hello world'), (err, compressed) => {
  // compressed[0] === 0x78 indicates valid deflate data
})

// Decompress
inflate(compressed, (err, result) => {
  result.toString() // 'hello world'
})
```

Also supports **Brotli** (`createBrotliCompress`/`createBrotliDecompress`, stable since v11.7) and experimental **Zstandard** (`ZstdCompress`/`ZstdDecompress`, v24+).

## File System

### API Surface Overview

| API Style | Methods | Use Case |
|-----------|---------|----------|
| **Promises (preferred)** | `fs/promises` — `readFile`, `writeFile`, `readdir`, etc. | Async/await workflows |
| Callbacks | `fs.readFile`, `fs.writeFile`, `fs.appendFile` | Legacy async code |
| Streaming | `createReadStream`, `createWriteStream` | Large files, pipe chains |
| POSIX low-level | `open`, `read`, `write`, `close`, `stat` | Fine-grained control |

`fs/promises` is the **modern, recommended API** (stable since v14). All POSIX and bulk methods also have `Sync` variants — **use sync only at startup**, never inside request handlers.

### File Descriptors

```typescript
import fs from 'node:fs'

const fd = fs.openSync('./data.txt', 'r')  // returns integer FD
const buf = Buffer.alloc(100)
fs.readSync(fd, buf, 0, 100, 0)            // read 100 bytes at offset 0
fs.closeSync(fd)
```

Standard FDs: `0` = stdin, `1` = stdout, `2` = stderr.

```typescript
fs.writeSync(1, 'direct to stdout\n')       // bypasses console.log
```

### Loading Config Files

```typescript
// Sync at startup — OK
const config = JSON.parse(fs.readFileSync('./config.json', 'utf8'))

// Or use require (cached globally — treat as frozen/read-only)
const config = require('./config.json')
```

### File Locking (Advisory)

Node has no built-in `flock`. Two strategies:

**Exclusive flag (`wx`)** — atomic, fails if file exists:

```typescript
fs.writeFile('config.lock', `${process.pid}`, { flag: 'wx' }, (err) => {
  if (err) return // lock held by another process
  // lock acquired — do work, then remove lockfile
})
```

**mkdir** — atomic, works on network drives:

```typescript
fs.mkdir('config.lock', (err) => {
  if (err) return // lock held
  fs.writeFileSync('config.lock/pid', `${process.pid}`)
  // do work, then fs.rmSync('config.lock', { recursive: true })
})
```

Clean up on exit:

```typescript
process.on('exit', () => {
  try { fs.unlinkSync('config.lock') } catch {}
})
```

### File Watching

Two APIs:

| Method | Backed by | Pros | Cons |
|--------|-----------|------|------|
| `fs.watch` | OS notifications (inotify/kqueue/FSEvents) | Fast, reliable | Inconsistent cross-platform |
| `fs.watchFile` | stat polling | Consistent, works on NFS | Slower, less event coverage |

```typescript
fs.watch('./dir', (eventType, filename) => {
  // eventType: 'rename' | 'change'
})

fs.watchFile('./file.txt', { interval: 1000 }, (curr, prev) => {
  if (curr.mtime !== prev.mtime) { /* file changed */ }
})
```

Prefer `fs.watch`; fall back to `fs.watchFile` for network drives. For production use, consider **chokidar** — it normalizes cross-platform inconsistencies, handles atomic saves, and supports reliable recursive watching on Linux.

### Recursive File Operations

Use `fs.readdir` with `{ recursive: true }` (Node 18.17+), or build manually:

```typescript
import { readdir, stat } from 'node:fs/promises'
import path from 'node:path'

const findFiles = async (dir: string, pattern: RegExp): Promise<string[]> => {
  const entries = await readdir(dir, { withFileTypes: true })
  const results: string[] = []
  for (const entry of entries) {
    const full = path.join(dir, entry.name)
    if (entry.isDirectory()) {
      results.push(...await findFiles(full, pattern))
    } else if (pattern.test(entry.name)) {
      results.push(full)
    }
  }
  return results
}
```

## Networking

### Network Layers Quick Reference

| Layer | Protocol | Node Module |
|-------|----------|-------------|
| Transport | TCP | `net` |
| Transport | UDP | `dgram` |
| Application | HTTP | `http` / `https` |
| Application | DNS | `dns` |
| Transport (secure) | TLS/SSL | `tls` |

### TCP (net module)

**Server:**

```typescript
import net from 'node:net'

let clientId = 0
const server = net.createServer((socket) => {
  const id = ++clientId
  socket.write(`Welcome, client ${id}\n`)
  socket.pipe(socket)  // echo — sockets are duplex streams

  socket.on('end', () => console.log(`Client ${id} disconnected`))
  socket.on('error', (err) => console.error(`Client ${id} error:`, err.message))
})

server.listen(8000)
```

**Client:**

```typescript
const client = net.connect(8000, 'localhost', () => {
  client.write('hello')
})
client.on('data', (data) => console.log(data.toString()))
client.on('error', (err) => console.error(err.message))
```

**Nagle's algorithm** — batches small packets for efficiency. Disable for low-latency apps:

```typescript
socket.setNoDelay(true) // send packets immediately (TCP_NODELAY)
```

### UDP (dgram module)

Connectionless, no delivery guarantees. Good for streaming, gaming, DNS.

**Server:**

```typescript
import dgram from 'node:dgram'

const server = dgram.createSocket('udp4')
server.on('message', (msg, rinfo) => {
  console.log(`${rinfo.address}:${rinfo.port} → ${msg}`)
  // Send response back using rinfo
  server.send('ack', rinfo.port, rinfo.address)
})
server.bind(41234)
```

**Client:**

```typescript
const client = dgram.createSocket('udp4')
const msg = Buffer.from('hello')
client.send(msg, 0, msg.length, 41234, 'localhost')
```

Max datagram payload: ~65,507 bytes (IPv4). Keep well below MTU (~1,400 bytes) to avoid silent drops.

### HTTP (http module)

`http.Server` extends `net.Server`. `http.createServer` is shorthand for `new http.Server`.

```typescript
import http from 'node:http'

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' })
  res.end('Hello\n')
})
server.listen(3000)
```

**Client request:**

```typescript
const req = http.request({ hostname: 'example.com', port: 80, path: '/' }, (res) => {
  res.on('data', (chunk) => process.stdout.write(chunk))
})
req.end()
```

Status code helpers: `http.STATUS_CODES[302]` → `'Found'`.

### DNS (dns module)

Two approaches:

```typescript
import dns from 'node:dns'

// dns.lookup — uses OS resolver (getaddrinfo), thread pool backed
dns.lookup('example.com', (err, address) => { /* single address */ })

// dns.resolve — uses c-ares, faster for bulk queries, returns array
dns.resolve('example.com', (err, addresses) => { /* ['93.184.216.34'] */ })
```

| Method | Returns |
|--------|---------|
| `dns.resolve` | A records (IPv4) |
| `dns.resolve6` | AAAA records (IPv6) |
| `dns.resolveTxt` | TXT records |
| `dns.resolveMx` | MX records |
| `dns.resolveSrv` | SRV records |
| `dns.resolveNs` | NS records |
| `dns.resolveCname` | CNAME records |

### TLS / HTTPS

`tls.Server` inherits from `net.Server`. Certificate setup:

```bash
# Generate CA key + cert
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days 365

# Generate server key + CSR + sign
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server-csr.pem
openssl x509 -req -in server-csr.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365
```

**TLS server:**

```typescript
import tls from 'node:tls'
import fs from 'node:fs'

const server = tls.createServer({
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
  ca: [fs.readFileSync('ca-cert.pem')],
  requestCert: true,
}, (socket) => {
  console.log('authorized:', socket.authorized)
  socket.write('secure hello\n')
  socket.pipe(socket)
})
server.listen(8000)
```

**HTTPS server:**

```typescript
import https from 'node:https'

const server = https.createServer({
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
}, (req, res) => {
  res.writeHead(200)
  res.end('secure\n')
})
server.listen(443)
```

## Key Architectural Insights

- **Non-blocking networking:** TCP/UDP use OS-level non-blocking I/O (epoll/kqueue). No thread pool.
- **File system:** Uses libuv thread pool internally, even though the API looks async.
- `net.Socket` is a duplex stream — all stream patterns (pipe, transform, backpressure) apply.
- `http.Server` is built on `net.Server` → `tls.Server` is built on `net.Server` → `https.Server` is built on `tls.Server`. Understanding this inheritance chain unlocks the full API surface.

For detailed examples, see [nodejs-networking-io-detail.md](references/nodejs-networking-io-detail.md).
