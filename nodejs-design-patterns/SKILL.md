---
name: nodejs-design-patterns
description: Reference guide for Node.js design patterns (creational, structural, behavioral). Use when writing, reviewing, or refactoring Node.js/TypeScript code and the user asks which pattern to apply, how to structure components, or when discussing Factory, Builder, Singleton, Proxy, Decorator, Adapter, Strategy, State, Template, Iterator, Middleware, Command, Revealing Constructor, or Dependency Injection patterns.
---

# Node.js Design Patterns Reference

Based on _Node.js Design Patterns_ (3rd Ed.) by Casciaro & Mammino.

In JavaScript, traditional GoF patterns are adapted to work with prototypal inheritance, closures, first-class functions, and dynamic typing. The implementation matters less than the problem each pattern solves.

## Quick Decision Guide

| Problem                                                               | Pattern                   | Category   |
| --------------------------------------------------------------------- | ------------------------- | ---------- |
| Decouple object creation from implementation                          | **Factory**               | Creational |
| Complex constructor with many params                                  | **Builder**               | Creational |
| Expose internals only during creation                                 | **Revealing Constructor** | Creational |
| Single shared instance across modules                                 | **Singleton**             | Creational |
| Decouple modules from their dependencies                              | **Dependency Injection**  | Creational |
| Control access to an object (validation, caching, logging, lazy init) | **Proxy**                 | Structural |
| Add new behavior to an existing object dynamically                    | **Decorator**             | Structural |
| Make an object compatible with a different interface                  | **Adapter**               | Structural |
| Swap algorithm/logic at runtime                                       | **Strategy**              | Behavioral |
| Change behavior based on internal state                               | **State**                 | Behavioral |
| Define algorithm skeleton, let subclasses fill steps                  | **Template**              | Behavioral |
| Uniform interface to traverse any collection                          | **Iterator**              | Behavioral |
| Modular processing pipeline (Express-style)                           | **Middleware**            | Behavioral |
| Encapsulate an action as an object (undo, queue, serialize)           | **Command**               | Behavioral |

## Creational Patterns

### Factory

Wrap `new` in a function. Decouples consumer from concrete class. Enables runtime type selection, encapsulation via closures, and smaller surface area.

```typescript
const createImage = (name: string) => {
  if (name.match(/\.jpe?g$/)) return new ImageJpeg(name);
  if (name.match(/\.png$/)) return new ImagePng(name);
  throw new Error("Unsupported format");
};
```

**When to use**: Object creation depends on runtime conditions. You want to hide classes behind a function. You need encapsulation via closures.

**In the wild**: Knex query builder, http.createServer, many npm packages export a factory.

### Builder

Fluent interface to construct complex objects step by step. Each method returns `this`. A final `build()` (or `invoke()`) produces the result.

```typescript
const boat = new BoatBuilder()
  .withMotors(2, "Brand", "Model")
  .withSails(1, "fabric", "white")
  .hullColor("blue")
  .build();
```

**When to use**: Constructor has many parameters. Related params should be grouped. You want self-documenting, guided creation.

**In the wild**: superagent HTTP client, Knex query chain.

### Revealing Constructor

Pass an `executor` function to the constructor that receives private internals. After construction, those internals are inaccessible.

```typescript
const immutable = new ImmutableBuffer(size, ({ write }) => {
  write("Hello!");
});
// immutable.write is undefined — only read methods exposed
```

**When to use**: Object should be modifiable only at creation time. You need initialization-only access to internals.

**In the wild**: `Promise` constructor (`resolve`, `reject` are revealed only to the executor).

### Singleton

Export a module-level instance. Node.js module cache ensures a single instance _per resolved path_.

```typescript
// db-instance.ts
export const db = new Database("my-app-db", { url: "localhost:5432" });
```

**Caveat**: Not a true singleton if the same package resolves to different paths in `node_modules` (version conflicts). Avoid `global` unless absolutely necessary. Prefer keeping packages stateless when shared as libraries.

### Dependency Injection

Pass dependencies as constructor/function arguments instead of hardcoding imports. The _injector_ (usually the entry point) wires everything together.

```typescript
// blog.ts — no import of db module
export class Blog {
  constructor(private db: Database) {}
  getAllPosts() {
    return this.db.all("SELECT * FROM posts");
  }
}

// index.ts — the injector
const db = createDb("data.sqlite");
const blog = new Blog(db);
```

**When to use**: You need testability (mock dependencies). A module should work with different backing implementations. You want loose coupling.

**Trade-off**: Harder to trace dependency graph at coding time. Can become unmanageable without a container (see awilix, inversify).

## Structural Patterns

### Proxy

Wraps a **subject** with an identical interface. Intercepts operations for validation, caching, logging, lazy init, or access control.

Three implementation techniques:

| Technique                              | Pros                                     | Cons                      |
| -------------------------------------- | ---------------------------------------- | ------------------------- |
| **Composition**                        | Safe, no mutation                        | Must delegate all methods |
| **Object augmentation** (monkey patch) | Simple, no delegation                    | Mutates subject           |
| **ES2015 `Proxy` object**              | No mutation, auto-delegation, trap-based | Cannot be polyfilled      |

```typescript
const safe = new Proxy(calculator, {
  get(target, prop) {
    if (prop === "divide") {
      return () => {
        if (target.peekValue() === 0) throw new Error("Division by 0");
        return target.divide();
      };
    }
    return target[prop];
  },
});
```

**When to use**: Data validation before forwarding. Authorization checks. Caching. Lazy initialization. Logging. Change observation (reactive programming).

**In the wild**: Vue 3 reactivity, MobX, LoopBack.

### Decorator

Augments an object with **new** functionality (unlike Proxy which only controls existing interface). Same implementation techniques as Proxy.

```typescript
const patchCalculator = (calc: Calculator) => {
  calc.add = () => {
    const a = calc.getValue(),
      b = calc.getValue();
    const result = a + b;
    calc.putValue(result);
    return result;
  };
  return calc;
};
```

**When to use**: Add capabilities to specific instances (not all instances of a class). Plugin systems. Wrapping third-party objects.

**In the wild**: LevelUP plugins, Fastify decorators, json-socket.

**Proxy vs Decorator**: Proxy controls access (same interface). Decorator adds behavior (extended interface). In JS, the line is blurry — treat them as complementary tools.

### Adapter

Wraps an **adaptee** to expose a different interface expected by the client. Pure interface translation.

```typescript
const createFsAdapter = (db: LevelDB) => ({
  readFile: (filename: string, cb: Callback) => {
    db.get(resolve(filename), { valueEncoding: "binary" }, (err, value) => {
      if (err) {
        if (err.notFound) cb(new Error("not found"));
        else cb(err);
      } else cb(null, value);
    });
  },
  writeFile: (filename: string, contents: Buffer, cb: Callback) => {
    db.put(resolve(filename), contents, { valueEncoding: "binary" }, cb);
  },
});
```

**When to use**: Integrate a component that has an incompatible interface. Swap storage backends. Cross-platform code (browser vs Node.js).

**In the wild**: LevelUP backends, database driver abstractions.

## Behavioral Patterns

### Strategy

Extract the _variable_ part of an algorithm into interchangeable **strategy** objects. The **context** holds the common logic.

```typescript
const iniStrategy = {
  deserialize: (data: string) => ini.parse(data),
  serialize: (data: object) => ini.stringify(data),
};

const config = new Config(iniStrategy); // swap to jsonStrategy at will
```

**When to use**: Multiple algorithms for the same task (serialization, auth, payment). Avoid complex if/else or switch blocks. Runtime algorithm selection.

**In the wild**: Passport.js authentication strategies.

### State

Like Strategy, but the strategy **changes dynamically** based on internal state. State transitions happen during the object's lifetime.

```typescript
class FailsafeSocket {
  constructor() {
    this.states = {
      offline: new OfflineState(this),
      online: new OnlineState(this),
    };
    this.changeState("offline");
  }
  changeState(state: string) {
    this.currentState = this.states[state];
    this.currentState.activate();
  }
  send(data: unknown) {
    this.currentState.send(data);
  }
}
```

**When to use**: Object behavior varies by state (connected/disconnected, pending/confirmed/cancelled). Replacing state-dependent if/else chains.

### Template

Base class defines the algorithm skeleton. Subclasses implement **template methods** (abstract steps).

```typescript
class ConfigTemplate {
  async load(file: string) {
    this.data = this._deserialize(await readFile(file, "utf-8"));
  }
  async save(file: string) {
    await writeFile(file, this._serialize(this.data));
  }
  _serialize(): never {
    throw new Error("Must implement");
  }
  _deserialize(): never {
    throw new Error("Must implement");
  }
}

class JsonConfig extends ConfigTemplate {
  _deserialize(data: string) {
    return JSON.parse(data);
  }
  _serialize(data: object) {
    return JSON.stringify(data);
  }
}
```

**When to use**: Family of components sharing structure but differing in specific steps. Prefer Strategy if you need runtime swapping.

**In the wild**: Node.js streams (`_read`, `_write`, `_transform`).

### Iterator

Implemented via **protocols** (not inheritance). An **iterator** has `next()` returning `{ value, done }`. An **iterable** has `[Symbol.iterator]()`. Async variants use `[Symbol.asyncIterator]()`.

```typescript
function* range(start: number, end: number) {
  for (let i = start; i <= end; i++) yield i;
}
for (const n of range(1, 5)) console.log(n);
```

**When to use**: Traverse any data structure uniformly. Lazy evaluation. Combine with `for...of`, destructuring, spread. Async iteration over streams or paginated APIs.

### Middleware

A processing pipeline where each function receives data, processes it, and calls `next()`. The Node.js incarnation of Chain of Responsibility.

```typescript
class MiddlewareManager {
  use(mw: Middleware) {
    this.pipeline.push(mw);
  }
  async execute(message: unknown) {
    let msg = message;
    for (const fn of this.pipeline) {
      msg = await fn(msg);
    }
    return msg;
  }
}
```

**When to use**: Plugin architecture. Request/response processing. Message transformation pipelines.

**In the wild**: Express, Koa, Fastify hooks.

### Command

Encapsulate an action as an object with `run()`, optionally `undo()` and `serialize()`. Enables deferred execution, undo, history, serialization over network.

```typescript
const createPostCmd = (service: StatusService, status: string) => {
  let postId: string | null = null;
  return {
    run() {
      postId = service.postUpdate(status);
    },
    undo() {
      if (postId) service.destroyUpdate(postId);
    },
    serialize() {
      return { type: "status", action: "post", status };
    },
  };
};
```

**When to use**: Undo/redo. Task queues. RPC / remote execution. Transaction batching. Operational transformation.

## Detailed References

For code examples and deeper implementation details, see:

- [creational-patterns.md](references/creational-patterns.md) — Factory, Builder, Revealing Constructor, Singleton, DI
- [structural-patterns.md](references/structural-patterns.md) — Proxy, Decorator, Adapter
- [behavioral-patterns.md](references/behavioral-patterns.md) — Strategy, State, Template, Iterator, Middleware, Command
