---
name: async-local-storage
description: AsyncLocalStorage - context propagation, request tracing, avoiding prop drilling
metadata:
  tags: async-local-storage, context, tracing, request-id, async_hooks
---

# AsyncLocalStorage

AsyncLocalStorage provides thread-local-storage semantics for async operations —
data stored inside a `run()` call is automatically available throughout the entire
async call chain without passing it as function arguments.

Stable since Node.js 16. Prefer it over raw `async_hooks.createHook` (which is
discouraged by Node.js docs due to usability, safety, and performance concerns).

## Core API

```javascript
const { AsyncLocalStorage } = require('node:async_hooks');

const als = new AsyncLocalStorage();

// Run a callback within a context — store is available to all async ops inside
als.run(store, callback, ...args);

// Get current store (returns undefined if called outside a run())
als.getStore();

// Run outside current context (store becomes undefined inside)
als.exit(callback, ...args);

// Enter context for the rest of the current synchronous execution
// Use with caution — bleeds into subsequent event handlers
als.enterWith(store);
```

## Classic use case: request-scoped context

```javascript
const http = require('node:http');
const { AsyncLocalStorage } = require('node:async_hooks');
const { randomUUID } = require('node:crypto');

const requestContext = new AsyncLocalStorage();

// Middleware: start context for each request
function contextMiddleware(req, res, next) {
  const store = new Map([
    ['requestId', randomUUID()],
    ['startTime', Date.now()],
    ['userId', null], // populated later by auth middleware
  ]);

  requestContext.run(store, next);
}

// Logger: no need to pass requestId around
function log(level, message, data = {}) {
  const store = requestContext.getStore();
  const requestId = store?.get('requestId') ?? 'no-context';
  console.log(JSON.stringify({ level, message, requestId, ...data }));
}

// Works anywhere in the call chain — service layer, repository, etc.
async function getUserById(id) {
  log('debug', 'fetching user', { id }); // requestId is automatically included
  return db.query('SELECT * FROM users WHERE id = $1', [id]);
}
```

## Context propagation is automatic through:

- `async/await`
- Promises (`.then`, `.catch`, `.finally`)
- `setTimeout` / `setInterval` / `setImmediate`
- `EventEmitter` callbacks
- `fs`, `net`, `http` callbacks

```javascript
als.run({ id: 42 }, () => {
  als.getStore(); // { id: 42 }

  setTimeout(() => {
    als.getStore(); // { id: 42 } — propagated through timer
  }, 100);

  Promise.resolve().then(() => {
    als.getStore(); // { id: 42 } — propagated through Promise
  });
});

als.getStore(); // undefined — outside run()
```

## Context is NOT propagated through:

```javascript
// Callbacks registered outside the run() scope
const emitter = new EventEmitter();

als.run({ id: 1 }, () => {
  // BAD: listener registered outside this run() won't get context
});

emitter.on('event', () => {
  als.getStore(); // undefined — listener was bound outside any context
});

// FIX: use AsyncResource.bind() for callbacks bound outside context
const { AsyncResource } = require('node:async_hooks');
const boundHandler = AsyncResource.bind(() => {
  als.getStore(); // captures context at bind time
});
```

## Storing mutable vs. immutable data

```javascript
// GOOD: immutable values — no mutation risk
als.run({ requestId: uuid(), userId: req.user.id }, callback);

// ACCEPTABLE: Map for request-scoped mutable state
als.run(new Map(), () => {
  als.getStore().set('authenticated', true);
});

// BAD: mutating a shared object leaks between requests
const sharedState = {}; // DON'T share across runs
als.run(sharedState, () => {
  sharedState.userId = 123; // pollutes all contexts using this object!
});
```

## Memory leak pitfalls

```javascript
// BAD: accessing ALS at module top level
// Modules are cached — this captures the context of the FIRST request
const store = als.getStore(); // undefined at import time, or wrong request context

// BAD: storing large objects or request references in store
als.run({ req, res, db, largePayload }, callback);
// These are kept alive for the full async duration of the request

// GOOD: store only what you need — small, serializable values
als.run({ requestId: uuid(), userId: req.user?.id }, callback);
```

## AsyncResource.bind — fix context loss in callbacks

```javascript
const { AsyncLocalStorage, AsyncResource } = require('node:async_hooks');
const EventEmitter = require('node:events');

const als = new AsyncLocalStorage();
const emitter = new EventEmitter();

als.run({ traceId: 'abc' }, () => {
  // Bind the handler to the current context at registration time
  emitter.on('data', AsyncResource.bind((chunk) => {
    console.log(als.getStore()?.traceId); // 'abc' — context preserved
  }));
});

emitter.emit('data', Buffer.from('hello'));
```

## snapshot() — capture context for later use

```javascript
// Capture current context as a callable
const snapshot = AsyncLocalStorage.snapshot();

// Later, in a different context, restore the captured one
snapshot(() => {
  als.getStore(); // returns the store that was active when snapshot() was called
});
```

## Performance notes

- Each `run()` has a small overhead (~microseconds) — fine for per-request use
- `getStore()` is fast (O(1) lookup in the async context chain)
- Avoid creating thousands of ALS instances — one per concern (auth, tracing, DB session)
- `enterWith()` is faster than `run()` but dangerous: bleeds context into later event handlers

## Common patterns

```javascript
// 1. Singleton per module (common pattern)
// context/requestContext.js
const { AsyncLocalStorage } = require('node:async_hooks');
module.exports = new AsyncLocalStorage();

// middleware.js
const ctx = require('./context/requestContext');
app.use((req, res, next) => ctx.run(new Map([['req', req]]), next));

// service.js — no req parameter needed!
const ctx = require('./context/requestContext');
function getCurrentUser() {
  return ctx.getStore()?.get('req')?.user;
}
```

```javascript
// 2. Distributed tracing — propagate trace ID through entire call graph
const traceStore = new AsyncLocalStorage();

function withTraceId(traceId, fn) {
  return traceStore.run({ traceId }, fn);
}

function getTraceId() {
  return traceStore.getStore()?.traceId;
}

// In HTTP client — pass trace ID to outgoing requests
function fetchWithTrace(url) {
  const traceId = getTraceId();
  return fetch(url, {
    headers: traceId ? { 'X-Trace-Id': traceId } : {},
  });
}
```

## References

- https://nodejs.org/api/async_context.html
- https://nodejs.org/api/async_hooks.html (createHook — discouraged)
