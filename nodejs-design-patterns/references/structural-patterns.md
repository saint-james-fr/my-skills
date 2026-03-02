# Structural Design Patterns — Detailed Reference

## Proxy

### Core Idea

An object with the **same interface** as the subject. Intercepts operations to add validation, caching, logging, lazy init, or access control — without changing the subject.

### Implementation: Object Composition

Wrap the subject. Must delegate every method explicitly.

```typescript
class SafeCalculator {
  constructor(private calculator: StackCalculator) {}

  divide() {
    if (this.calculator.peekValue() === 0) throw new Error('Division by 0')
    return this.calculator.divide()
  }

  putValue(v: number)  { return this.calculator.putValue(v) }
  getValue()           { return this.calculator.getValue() }
  peekValue()          { return this.calculator.peekValue() }
  clear()              { return this.calculator.clear() }
  multiply()           { return this.calculator.multiply() }
}
```

### Implementation: Object Augmentation (Monkey Patching)

Replace methods on the subject directly. Simple but **mutates** the original — dangerous if the subject is shared.

```typescript
const patchToSafe = (calc: StackCalculator) => {
  const divideOrig = calc.divide
  calc.divide = () => {
    if (calc.peekValue() === 0) throw new Error('Division by 0')
    return divideOrig.apply(calc)
  }
  return calc
}
```

### Implementation: ES2015 Proxy Object

No mutation, automatic delegation. The `get` trap intercepts property access.

```typescript
const handler: ProxyHandler<StackCalculator> = {
  get(target, prop) {
    if (prop === 'divide') {
      return () => {
        if (target.peekValue() === 0) throw new Error('Division by 0')
        return target.divide()
      }
    }
    return (target as any)[prop]
  }
}

const safe = new Proxy(calculator, handler)
// safe instanceof StackCalculator === true
```

Available traps: `get`, `set`, `has`, `deleteProperty`, `apply`, `construct`, and more.

### Proxy for Change Observation (Reactive Pattern)

```typescript
const createObservable = <T extends object>(
  target: T,
  observer: (change: { prop: string; prev: unknown; curr: unknown }) => void
) => new Proxy(target, {
  set(obj, prop, value) {
    if (value !== (obj as any)[prop]) {
      const prev = (obj as any)[prop]
      ;(obj as any)[prop] = value
      observer({ prop: prop as string, prev, curr: value })
    }
    return true
  }
})
```

Cornerstone of reactive programming. Used by Vue 3 and MobX.

### Logging Writable Stream (Real-World Example)

```typescript
const createLoggingWritable = (writable: Writable) =>
  new Proxy(writable, {
    get(target, prop) {
      if (prop === 'write') {
        return (...args: unknown[]) => {
          console.log('Writing', args[0])
          return writable.write(...(args as Parameters<typeof writable.write>))
        }
      }
      return (target as any)[prop]
    }
  })
```

### Choosing a Technique

| Need | Best Technique |
|------|----------------|
| Safety (no subject mutation) | Composition or `Proxy` object |
| Minimal code (few methods to intercept) | Object augmentation |
| Dynamic property interception, meta-programming | `Proxy` object |
| Lazy initialization | Composition |

---

## Decorator

### Core Idea

Add **new functionality** to a specific instance (not the class). Same techniques as Proxy, different intent.

| Proxy | Decorator |
|-------|-----------|
| Same interface, controls access | Extended interface, adds behavior |
| Does not add new methods | Adds new methods |

In JS, the line is blurry — treat them as complementary.

### Composition

```typescript
class EnhancedCalculator {
  constructor(private calculator: StackCalculator) {}

  // NEW method
  add() {
    const b = this.calculator.getValue()
    const a = this.calculator.getValue()
    const result = a + b
    this.calculator.putValue(result)
    return result
  }

  // ENHANCED existing method
  divide() {
    if (this.calculator.peekValue() === 0) throw new Error('Division by 0')
    return this.calculator.divide()
  }

  // Delegated
  putValue(v: number) { return this.calculator.putValue(v) }
  getValue()          { return this.calculator.getValue() }
  peekValue()         { return this.calculator.peekValue() }
  clear()             { return this.calculator.clear() }
  multiply()          { return this.calculator.multiply() }
}
```

### Object Augmentation (Plugin Pattern)

Most pragmatic for Node.js plugin systems. Attach new methods directly.

```typescript
const levelSubscribe = (db: LevelDB) => {
  db.subscribe = (pattern: Record<string, unknown>, listener: Function) => {
    db.on('put', (key: string, val: Record<string, unknown>) => {
      const isMatch = Object.keys(pattern).every(k => pattern[k] === val[k])
      if (isMatch) listener(key, val)
    })
  }
  return db
}

levelSubscribe(db)
db.subscribe({ doctype: 'tweet', language: 'en' }, (k, v) => console.log(v))
```

### With ES2015 Proxy

```typescript
const handler: ProxyHandler<StackCalculator> = {
  get(target, prop) {
    if (prop === 'add') {
      return () => {
        const b = target.getValue(), a = target.getValue()
        const result = a + b
        target.putValue(result)
        return result
      }
    }
    return (target as any)[prop]
  }
}
```

---

## Adapter

### Core Idea

Wrap an **adaptee** to expose a completely different interface. No new behavior — pure interface translation.

### LevelDB as Filesystem

```typescript
import { resolve } from 'path'

const createFsAdapter = (db: LevelDB) => ({
  readFile(filename: string, options: any, cb: Function) {
    if (typeof options === 'function') { cb = options; options = {} }
    db.get(resolve(filename), { valueEncoding: 'binary' }, (err, value) => {
      if (err) {
        if (err.notFound) return cb(new Error(`ENOENT: ${filename}`))
        return cb(err)
      }
      cb(null, value)
    })
  },

  writeFile(filename: string, contents: Buffer, options: any, cb: Function) {
    if (typeof options === 'function') { cb = options; options = {} }
    db.put(resolve(filename), contents, { valueEncoding: 'binary' }, cb)
  }
})

const fs = createFsAdapter(db)
fs.writeFile('file.txt', Buffer.from('Hello'), () => {
  fs.readFile('file.txt', (err, data) => console.log(data))
})
```

### Strategy vs Adapter

- **Strategy**: Context and strategy both implement logic; they combine to form the full algorithm
- **Adapter**: Only translates interface; doesn't add algorithmic logic

### Cross-Platform Use Cases

- Swap `localStorage` for a database in browser vs server
- Wrap a REST API behind a local interface
- Unify different database drivers under one interface
