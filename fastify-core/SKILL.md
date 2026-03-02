---
name: fastify-core
description: Reference guide for Fastify framework fundamentals — plugin system, encapsulation, hooks lifecycle, routes, async handler pitfalls, decorators, validation, and serialization. Use when building Fastify apps, writing plugins, adding routes/hooks, debugging encapsulation issues, configuring validation with JSON Schema, or understanding the request lifecycle.
---

# Fastify Core Reference

Based on _Accelerating Server-Side Development with Fastify_ (2023), Chapters 1-5.

## Components Overview

| Component                     | Role                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------- |
| **Root application instance** | Main API, wraps `http.Server`, manages routes & plugins                         |
| **Plugin instance**           | Child of root, isolated encapsulated context                                    |
| **Request**                   | Wrapper around `http.IncomingMessage` — `body`, `params`, `query`, `headers`    |
| **Reply**                     | Wrapper around `http.ServerResponse` — `send()`, `code()`, `header()`, `type()` |
| **Hooks**                     | Functions that run at specific lifecycle events                                 |
| **Decorators**                | Extend Request, Reply, or instance with custom properties                       |

## Plugin System & Encapsulation

### Creating a Plugin

```typescript
app.register(
  async function myPlugin(fastify, opts) {
    fastify.get("/hello", async () => "world");
  },
  { prefix: "/api" },
);
```

A plugin receives a **new Fastify instance** (child context) that inherits from the parent. Everything added inside the plugin stays inside that context and its children.

### Encapsulation Rules

- **Decorators, hooks, plugins, routes** are all scoped to their encapsulated context
- A child inherits from parent, but parent cannot see child's additions
- Siblings cannot see each other's additions

```typescript
app.decorate("db", dbConnection); // visible everywhere
app.register(async function pluginA(instance) {
  instance.decorate("cache", new Map()); // only visible in pluginA and its children
  instance.get("/data", handler); // inherits app's 'db' decorator
});
app.register(async function pluginB(instance) {
  // instance.cache → undefined (sibling isolation)
});
```

### Breaking Encapsulation with `fastify-plugin`

When a plugin needs to **decorate the parent** (e.g., DB connection, auth), wrap it with `fastify-plugin`:

```typescript
import fp from "fastify-plugin";

export default fp(
  async function dbPlugin(fastify, opts) {
    const client = await connectToDb(opts.connectionString);
    fastify.decorate("db", client);
    fastify.addHook("onClose", async () => client.close());
  },
  {
    name: "db-plugin",
    fastify: "4.x",
    decorators: { fastify: [] },
    dependencies: [],
  },
);
```

**When to use `fp()`**: shared services (DB, cache, auth).
**When NOT to use `fp()`**: route plugins, feature modules — keep them encapsulated.

### Boot Sequence

1. Plugins loaded in registration order (async, deterministic)
2. `onRoute` / `onRegister` hooks fire synchronously during registration
3. `onReady` hooks run after all plugins loaded, before `listen()`
4. Server starts accepting requests
5. `onClose` hooks run on `app.close()`

**`ERR_AVVIO_PLUGIN_TIMEOUT`**: Most common boot error. Forgot to call `done()` in callback-style plugin, or Promise never resolved. Default timeout: 10s (configurable via `pluginTimeout`).

### Declaration Order Best Practice

Inside each plugin context, register in this order:

1. External npm plugins
2. Your custom plugins
3. Decorators
4. Hooks
5. Routes

## Routes

### Declaration Styles

```typescript
// Full options
app.route({ method: "GET", url: "/hello", handler });

// Shorthand
app.get("/hello", handler);
app.get("/hello", { schema }, handler);
```

### Async vs Sync Handlers

|                                    | Async handler                    | Sync handler                |
| ---------------------------------- | -------------------------------- | --------------------------- |
| Signature                          | `async function(request, reply)` | `function(request, reply)`  |
| How to reply                       | `return payload`                 | `reply.send(payload)`       |
| How to error                       | `throw error`                    | `reply.send(errorInstance)` |
| If calling `reply.send()` in async | Must `return reply`              | Fine as-is                  |

**Critical rule**: Never call `reply.send()` more than once.

```typescript
// PREFERRED: async + return
app.get("/hello", async (request, reply) => {
  const data = await fetchData();
  return { data };
});

// If you must use reply.send() in async handler:
async function handler(request, reply) {
  legacySendHandler(request, reply);
  return reply; // tell Fastify reply is handled elsewhere
}
```

### Named Functions Over Arrow Functions

Use named functions for handlers and hooks — you get:

- `this` bound to the Fastify instance (access `this.db`, `this.log`)
- Better stack traces and `printRoutes()` output

```typescript
app.get("/data", function getData(request, reply) {
  return this.db.query("SELECT * FROM items");
});
```

### Error Handling

```typescript
// Custom error handler (encapsulated)
app.setErrorHandler(async function(error, request, reply) {
  request.log.error(error)
  reply.code(error.statusCode ?? 500)
  return { error: error.message }
})

// Route-level error handler
app.get('/route', {
  handler: myHandler,
  errorHandler: async function(error, request, reply) { ... }
})

// Custom 404
app.setNotFoundHandler(async function(request, reply) {
  reply.code(404)
  return { message: 'Not found' }
})
```

### Parametric Routes & Constraints

```typescript
app.get("/users/:userId/posts/:postId", handler);
app.get("/user", { constraints: { version: "2.0.0" } }, handlerV2);
app.get("/user", handlerV1); // no constraint = default
```

**Route priority**: exact match → param → wildcard → param+regex.

## Hooks Lifecycle

### Application Hooks

| Hook         | When                             | Sync/Async | Encapsulated |
| ------------ | -------------------------------- | ---------- | ------------ |
| `onRoute`    | Route registered                 | Sync       | Yes          |
| `onRegister` | New encapsulated context created | Sync       | Yes          |
| `onReady`    | Before server starts listening   | Async      | No           |
| `onClose`    | On `app.close()`                 | Async      | Yes          |

### Request/Reply Hooks (in order)

| Hook               | Purpose                                          |
| ------------------ | ------------------------------------------------ |
| `onRequest`        | First hook after routing (auth, rate limiting)   |
| `preParsing`       | Before body parsing (modify raw stream)          |
| `preValidation`    | Before JSON Schema validation                    |
| `preHandler`       | After validation, before handler (authorization) |
| `preSerialization` | Before response serialization                    |
| `onSend`           | Last chance to modify response payload           |
| `onResponse`       | After response sent (logging, metrics)           |
| `onError`          | Only on error (any phase)                        |
| `onTimeout`        | Only on timeout (any phase)                      |

```typescript
app.addHook("onRequest", async function auth(request, reply) {
  const token = request.headers.authorization;
  if (!token) throw { statusCode: 401, message: "Unauthorized" };
});
```

For detailed examples, see [fastify-core-detail.md](references/fastify-core-detail.md).

```

```
