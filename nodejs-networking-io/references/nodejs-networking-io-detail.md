# Node.js Networking & I/O — Detail Reference

## Parsing a Binary File Format (DBase 5.0 Example)

Step-by-step pattern for reading any binary format:

### 1. Read the spec, map bytes to Buffer methods

```typescript
import fs from "node:fs";

const buf = fs.readFileSync("./data.dbf");

// Byte 0: version (1 byte)
const version = buf[0];

// Bytes 1-3: date (YYMMDD, each byte is a number)
const date = new Date();
date.setFullYear(1900 + buf[1]);
date.setMonth(buf[2]);
date.setDate(buf[3]);

// Bytes 4-7: record count (32-bit unsigned LE)
const totalRecords = buf.readUInt32LE(4);

// Bytes 8-9: header size (16-bit unsigned LE)
const headerSize = buf.readUInt16LE(8);

// Bytes 10-11: record size (16-bit unsigned LE)
const recordSize = buf.readUInt16LE(10);
```

### 2. Parse repeating field descriptors

```typescript
const fields: Array<{ name: string; type: string; length: number }> = [];
let offset = 32;
const FIELD_TERMINATOR = 0x0d;

while (buf[offset] !== FIELD_TERMINATOR) {
  const fieldBuf = buf.subarray(offset, offset + 32); // view, not copy

  const name = fieldBuf.toString("ascii", 0, 11).replace(/\u0000/g, "");
  const type = fieldBuf.toString("ascii", 11, 12) === "N" ? "Numeric" : "Character";
  const length = fieldBuf[16];

  fields.push({ name, type, length });
  offset += 32;
}
```

### 3. Read records using field metadata

```typescript
const records: Record<string, unknown>[] = [];

for (let i = 0; i < totalRecords; i++) {
  let recordOffset = headerSize + i * recordSize;
  const isDeleted = buf[recordOffset] === 0x2a;
  recordOffset++; // skip deletion flag

  if (isDeleted) continue;

  const record: Record<string, unknown> = {};
  for (const field of fields) {
    const raw = buf.toString("ascii", recordOffset, recordOffset + field.length).trim();
    record[field.name] = field.type === "Numeric" ? Number(raw) : raw;
    recordOffset += field.length;
  }
  records.push(record);
}
```

### Key Buffer Method Reference

| Method                         | Bytes | Endianness             |
| ------------------------------ | ----- | ---------------------- |
| `readUInt8(offset)`            | 1     | —                      |
| `readUInt16LE(offset)`         | 2     | Little-endian          |
| `readUInt16BE(offset)`         | 2     | Big-endian             |
| `readUInt32LE(offset)`         | 4     | Little-endian          |
| `readInt8(offset)`             | 1     | — (signed)             |
| `readInt16LE(offset)`          | 2     | Little-endian (signed) |
| `readInt32LE(offset)`          | 4     | Little-endian (signed) |
| `readFloatLE(offset)`          | 4     | Little-endian          |
| `readDoubleLE(offset)`         | 8     | Little-endian          |
| `writeUInt8(value, offset)`    | 1     | —                      |
| `writeUInt16LE(value, offset)` | 2     | Little-endian          |
| `writeUInt32LE(value, offset)` | 4     | Little-endian          |

## Custom Binary Protocol Design

### Protocol spec example

| Byte | Contents                 | Purpose                           |
| ---- | ------------------------ | --------------------------------- |
| 0    | 1 byte bitmask           | Which databases (1-8) to write to |
| 1    | 1 byte uint              | Key (0-255)                       |
| 2+   | variable (zlib deflated) | Compressed value data             |

### Encoding a message

```typescript
import { deflateSync } from "node:zlib";

const databases = 0b00000101; // databases 1 and 3
const key = 42;

const compressed = deflateSync(Buffer.from("hello world"));

const message = Buffer.alloc(2 + compressed.length);
message.writeUInt8(databases, 0);
message.writeUInt8(key, 1);
compressed.copy(message, 2);
```

### Decoding a message

```typescript
import { inflateSync } from "node:zlib";

const dbByte = message.readUInt8(0);
const key = message.readUInt8(1);

const bitmasks = [1, 2, 4, 8, 16, 32, 64, 128];
const targetDbs = bitmasks.map((mask, i) => ((dbByte & mask) === mask ? i : -1)).filter((i) => i >= 0);

// Verify zlib magic byte
if (message[2] === 0x78) {
  const value = inflateSync(message.subarray(2)).toString();
  for (const db of targetDbs) {
    database[db][key] = value;
  }
}
```

## Append-Only File Database Pattern

In-memory state + append-only journal on disk:

```typescript
import fs from "node:fs";
import { EventEmitter } from "node:events";

class Database extends EventEmitter {
  private store = new Map<string, unknown>();
  private fd: number;

  constructor(private path: string) {
    super();
    this.fd = fs.openSync(path, "a+");
    this.load();
  }

  private load() {
    const stream = fs.createReadStream(this.path, { encoding: "utf8" });
    let remainder = "";

    stream.on("data", (chunk: string) => {
      const lines = (remainder + chunk).split("\n");
      remainder = lines.pop()!;
      for (const line of lines) {
        if (!line) continue;
        const { key, value } = JSON.parse(line);
        if (value === null) this.store.delete(key);
        else this.store.set(key, value);
      }
    });

    stream.on("end", () => this.emit("load"));
  }

  get(key: string) {
    return this.store.get(key);
  }

  set(key: string, value: unknown) {
    this.store.set(key, value);
    fs.writeSync(this.fd, JSON.stringify({ key, value }) + "\n");
  }

  del(key: string) {
    this.store.delete(key);
    fs.writeSync(this.fd, JSON.stringify({ key, value: null }) + "\n");
  }
}
```

Properties: efficient disk I/O (append-only), durable (old data never overwritten), trivial backups (copy the file).

## TCP Echo Server with Client Tracking

```typescript
import net from "node:net";
import { strict as assert } from "node:assert";

let clientId = 0;

const server = net.createServer((socket) => {
  const id = ++clientId;
  socket.write(`client ${id}\n`);
  socket.pipe(socket);
  socket.on("error", () => {});
});

server.listen(8000, () => {
  // In-process test client
  const client = net.connect(8000, () => {
    client.on("data", (data) => {
      assert.equal(data.toString(), "client 1\n");
      client.end();
    });
  });
  client.on("end", () => server.close());
});
```

TCP clients and servers can run in-process — useful for testing.

## HTTP Redirect Follower

```typescript
import http from "node:http";
import https from "node:https";
import { URL } from "node:url";

const MAX_REDIRECTS = 10;

const fetch = (urlStr: string, redirects = 0): Promise<string> => {
  return new Promise((resolve, reject) => {
    if (redirects >= MAX_REDIRECTS) return reject(new Error("Too many redirects"));

    const url = new URL(urlStr);
    const client = url.protocol === "https:" ? https : http;

    client
      .get(url, (res) => {
        if (res.statusCode && res.statusCode >= 300 && res.statusCode < 400 && res.headers.location) {
          return resolve(fetch(res.headers.location, redirects + 1));
        }

        let body = "";
        res.on("data", (chunk) => (body += chunk));
        res.on("end", () => resolve(body));
      })
      .on("error", reject);
  });
};
```

Key insight: check protocol on each redirect — servers often redirect HTTP → HTTPS.

## HTTP Proxy

```typescript
import http from "node:http";
import { URL } from "node:url";

const proxy = http.createServer((clientReq, clientRes) => {
  const target = new URL(clientReq.url!);
  const options = {
    hostname: target.hostname,
    port: target.port || 80,
    path: target.pathname + target.search,
    method: clientReq.method,
    headers: clientReq.headers,
  };

  const proxyReq = http.request(options, (proxyRes) => {
    clientRes.writeHead(proxyRes.statusCode!, proxyRes.headers);
    proxyRes.pipe(clientRes);
  });

  clientReq.pipe(proxyReq);
  proxyReq.on("error", (err) => {
    clientRes.writeHead(502);
    clientRes.end(`Proxy error: ${err.message}`);
  });
});

proxy.listen(8080);
```

## DNS: lookup vs resolve

|              | `dns.lookup`                  | `dns.resolve`        |
| ------------ | ----------------------------- | -------------------- |
| Backend      | OS getaddrinfo (thread pool)  | c-ares library       |
| Returns      | Single address                | Array of addresses   |
| Consistency  | Matches OS behavior           | Independent resolver |
| Speed (bulk) | Slower (thread pool limited)  | Faster               |
| Used by      | `net.connect`, `http.request` | Direct queries       |

```typescript
import dns from "node:dns";

// lookup — single result, OS-consistent
dns.lookup("example.com", (err, address, family) => {
  // address: '93.184.216.34', family: 4
});

// resolve — array, supports record types
dns.resolve4("example.com", (err, addresses) => {
  // addresses: ['93.184.216.34']
});

dns.resolveMx("example.com", (err, records) => {
  // records: [{ exchange: 'mail.example.com', priority: 10 }]
});
```

## TLS Certificate Generation Cheat Sheet

```bash
# 1. Create Certificate Authority (CA)
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days 365 \
  -subj "/CN=MyCA"

# 2. Create server key + CSR (Common Name = hostname)
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server-csr.pem \
  -subj "/CN=$(hostname)"

# 3. Sign server cert with CA
openssl x509 -req -in server-csr.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365

# 4. Create client key + CSR + sign (for mutual TLS)
openssl genrsa -out client-key.pem 2048
openssl req -new -key client-key.pem -out client-csr.pem \
  -subj "/CN=client"
openssl x509 -req -in client-csr.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out client-cert.pem -days 365
```

**Common Name must match `servername`** in `tls.connect()` options.

## Testing TLS with OpenSSL CLI

```bash
# Test your TLS server
openssl s_client -connect 127.0.0.1:8000 -CAfile ./ca-cert.pem

# Start a test TLS server
openssl s_server -key server-key.pem -cert server-cert.pem -accept 8000
```
