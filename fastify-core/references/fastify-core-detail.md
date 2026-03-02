# Fastify Core — Detailed Reference

## Plugin Prefix & Route Reuse

Register the same route plugin with different prefixes for API versioning:

```typescript
import usersRouter from "./routes/users";

app.register(usersRouter, { prefix: "/v1" });
app.register(
  async function v2(instance) {
    instance.register(usersRouter);
    instance.delete("/users/:name", deleteHandler);
  },
  { prefix: "/v2" },
);
```

Result: `GET /v1/users`, `GET /v2/users`, `DELETE /v2/users/:name`.

## Encapsulation Deep Dive

### What Gets Scoped

```
Root (app)
├── decorator: db ✓ (via fastify-plugin, visible everywhere)
├── hook: parseUser (runs for ALL routes)
├── GET /public (hooks: [parseUser])
│
├── Plugin: admin (prefix: /admin)
│   ├── hook: requireAdmin (only runs for /admin/* routes)
│   ├── GET /admin/dashboard (hooks: [parseUser, requireAdmin])
│   │
│   └── Plugin: superAdmin
│       ├── hook: requireSuperAdmin
│       └── GET /admin/system (hooks: [parseUser, requireAdmin, requireSuperAdmin])
│
└── Plugin: api (prefix: /api)
    ├── GET /api/items (hooks: [parseUser] — no admin hooks!)
    └── ...
```

### Rule of Thumb

- `fastify-plugin` = **shared infrastructure** (DB, auth service, decorators)
- Raw plugin (no fp wrapper) = **feature module** (routes, business logic)

## Request/Reply Hook Signatures

```typescript
// Most hooks: (request, reply) → can short-circuit by sending reply
app.addHook('onRequest', async (request, reply) => { ... })
app.addHook('preHandler', async (request, reply) => { ... })

// preParsing: receives raw stream payload
app.addHook('preParsing', async (request, reply, payload) => {
  return modifiedPayload  // must return stream/buffer
})

// preSerialization: receives the response payload before stringify
app.addHook('preSerialization', async (request, reply, payload) => {
  return { ...payload, timestamp: Date.now() }
})

// onSend: receives the serialized string/buffer
app.addHook('onSend', async (request, reply, payload) => {
  return payload  // can modify the final string
})

// onResponse: after response sent (no reply modification possible)
app.addHook('onResponse', async (request, reply) => {
  request.log.info({ responseTime: reply.elapsedTime }, 'request completed')
})

// onError: only on error
app.addHook('onError', async (request, reply, error) => {
  request.log.error(error)
})
```

### Short-Circuiting from Hooks

Any hook before the handler (`onRequest` through `preHandler`) can short-circuit the request by sending a reply:

```typescript
app.addHook("onRequest", async (request, reply) => {
  if (!request.headers.authorization) {
    reply.code(401).send({ error: "Unauthorized" });
    return; // short-circuit: handler never runs
  }
});
```

## Error Handler Cascade

Error handlers are encapsulated. When a route-level handler re-throws, it bubbles to the parent:

```
Route errorHandler → Plugin errorHandler → Parent errorHandler → Root default
```

```typescript
app.register(async function apiPlugin(instance) {
  instance.setErrorHandler(async (error, request, reply) => {
    if (error.code === "KNOWN_ERROR") {
      reply.code(400);
      return { error: error.message };
    }
    throw error; // re-throw to parent error handler
  });

  instance.register(async function childPlugin(child) {
    child.setErrorHandler(async (error, request, reply) => {
      // most specific handler — runs first for child routes
      throw error; // pass to apiPlugin's error handler
    });
    child.get("/fail", async () => {
      throw new Error("ops");
    });
  });
});
```

## Decorators

Add custom properties to Fastify instance, Request, or Reply:

```typescript
// Instance decorator
app.decorate("config", loadConfig());
app.decorate("db", dbConnection);

// Request decorator (per-request state)
app.decorateRequest("user", null);
app.addHook("onRequest", async (request) => {
  request.user = await getUserFromToken(request.headers.authorization);
});

// Reply decorator
app.decorateReply("sendSuccess", function (data) {
  this.code(200).send({ success: true, data });
});
```

**Important**: `decorateRequest` / `decorateReply` must set the initial value to `null` (not an object reference), because Fastify reuses the prototype for performance.

## Content Type Parsing

Fastify handles `application/json` and `text/plain` by default. Add custom parsers:

```typescript
app.addContentTypeParser("application/xml", { parseAs: "string" }, async (request, body) => {
  return xmlParser.parse(body);
});

// Catch-all parser
app.addContentTypeParser("*", async (request, payload) => {
  const data = await collectStream(payload);
  return data;
});
```

## Router Internals

- Uses `find-my-way` (Radix tree) for O(log n) routing
- Routes compiled at startup — cannot add routes after `ready()`
- Route matching order: exact → parametric → wildcard → parametric+regex
- `rewriteUrl` option allows URL rewriting before routing

## `app.inject()` — Testing Without Network

```typescript
const response = await app.inject({
  method: "GET",
  url: "/hello",
  headers: { authorization: "Bearer token123" },
});

console.log(response.statusCode); // 200
console.log(response.json()); // { message: 'world' }
```

No network needed — runs the full request lifecycle in-process.
