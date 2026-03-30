---
name: events
description: EventEmitter internals, memory leaks, error handling, async events
metadata:
  tags: events, eventemitter, memory-leaks, error-handling, patterns
---

# Node.js EventEmitter

## Core API

```javascript
const { EventEmitter } = require('node:events');

const emitter = new EventEmitter();

emitter.on('event', handler);        // add persistent listener
emitter.once('event', handler);      // add one-shot listener (auto-removes after first emit)
emitter.off('event', handler);       // remove specific listener (alias: removeListener)
emitter.emit('event', arg1, arg2);   // synchronous — all listeners run before emit returns
emitter.removeAllListeners('event'); // remove all listeners for event
emitter.listeners('event');          // array of listener functions
emitter.listenerCount('event');      // number of listeners
emitter.eventNames();                // all events with registered listeners
```

## The `error` event is special

If `'error'` is emitted with no listener, Node.js **throws** and the process may crash:

```javascript
// BAD: unhandled error event crashes process
const emitter = new EventEmitter();
emitter.emit('error', new Error('boom')); // throws!

// GOOD: always attach an error handler
emitter.on('error', (err) => {
  console.error('Emitter error:', err);
});

// GOOD: use errorMonitor to observe without consuming the error event
const { errorMonitor } = require('node:events');
emitter.on(errorMonitor, (err) => {
  metrics.increment('emitter.error');
  // error still propagates to 'error' listeners (or throws if none)
});
```

## maxListeners — the leak warning

Default limit is **10 listeners per event**. Exceeding it prints a warning:

```
MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
11 data listeners added to [EventEmitter].
```

This is a **warning, not an error** — the event still fires.

```javascript
// Silence legitimate use of many listeners
emitter.setMaxListeners(100);

// Global default change
EventEmitter.defaultMaxListeners = 20;

// Silence for a single operation
emitter.setMaxListeners(emitter.getMaxListeners() + 1);
someOperation();
emitter.setMaxListeners(emitter.getMaxListeners() - 1);
```

## Listener memory leaks — most common source

```javascript
// LEAK: re-registering listeners on each request
app.get('/stream', (req, res) => {
  dataEmitter.on('data', (chunk) => res.write(chunk)); // never removed!
  dataEmitter.on('end', () => res.end());               // never removed!
});

// FIX: remove listeners when the consumer is done
app.get('/stream', (req, res) => {
  function onData(chunk) { res.write(chunk); }
  function onEnd() {
    res.end();
    cleanup();
  }
  function onReqClose() { cleanup(); } // client disconnected

  function cleanup() {
    dataEmitter.off('data', onData);
    dataEmitter.off('end', onEnd);
    req.off('close', onReqClose);
  }

  dataEmitter.on('data', onData);
  dataEmitter.on('end', onEnd);
  req.on('close', onReqClose); // always handle client disconnect
});

// SIMPLER FIX: use once() + AbortSignal
app.get('/stream', (req, res) => {
  const ac = new AbortController();
  req.on('close', () => ac.abort());
  events.on(dataEmitter, 'data', { signal: ac.signal })
    .then(async (iter) => {
      for await (const [chunk] of iter) res.write(chunk);
      res.end();
    })
    .catch(() => res.end());
});
```

## emit is synchronous — don't block it

```javascript
// All listeners run synchronously and sequentially in registration order
emitter.on('data', (buf) => {
  heavyCPUWork(buf); // blocks subsequent listeners AND the caller
});

// GOOD: defer heavy work
emitter.on('data', (buf) => {
  setImmediate(() => heavyCPUWork(buf)); // returns control immediately
});

// GOOD: async listeners — but emit() won't await them!
emitter.on('data', async (buf) => {
  await saveToDb(buf); // errors here are UNHANDLED REJECTIONS
});

// FIX: wrap async listeners to handle errors
emitter.on('data', (buf) => {
  saveToDb(buf).catch((err) => emitter.emit('error', err));
});
```

## events.once — promisify a one-shot event

```javascript
const { once } = require('node:events');

// Wait for a single event (with cleanup on abort)
const ac = new AbortController();
setTimeout(() => ac.abort(), 5000); // timeout

try {
  const [data] = await once(emitter, 'data', { signal: ac.signal });
  console.log('Got:', data);
} catch (err) {
  if (err.name === 'AbortError') console.log('Timed out');
  else throw err;
}
```

## events.on — async iterator over events

```javascript
const { on } = require('node:events');

const ac = new AbortController();

// Consume events as async iterable (useful for streams of events)
async function consume() {
  for await (const [chunk] of on(emitter, 'data', { signal: ac.signal })) {
    process(chunk);
  }
}

// Stop consuming
ac.abort();
```

## Extending EventEmitter

```javascript
class Database extends EventEmitter {
  constructor(url) {
    super();
    this.url = url;
  }

  async connect() {
    try {
      this.connection = await createConnection(this.url);
      this.emit('connect', this.connection);
    } catch (err) {
      this.emit('error', err); // callers must handle 'error'
    }
  }

  async query(sql, params) {
    const result = await this.connection.execute(sql, params);
    this.emit('query', { sql, params, rows: result.length });
    return result;
  }

  close() {
    this.connection?.end();
    this.emit('close');
    this.removeAllListeners(); // prevent leaks when object is discarded
  }
}

const db = new Database(url);
db.on('error', console.error);
db.on('query', ({ sql, rows }) => metrics.record(sql, rows));
await db.connect();
```

## captureRejections — automatic async error routing

```javascript
// Node.js 12+: async listener errors auto-emit to 'error'
const emitter = new EventEmitter({ captureRejections: true });

emitter.on('data', async (chunk) => {
  await riskyOperation(chunk); // rejection → emits 'error' automatically
});

emitter.on('error', (err) => console.error('Caught:', err));

// Enable globally
EventEmitter.captureRejections = true;
```

## Listener ordering and prepend

```javascript
emitter.on('event', handlerA);          // added to end
emitter.on('event', handlerB);          // added to end — fires after A
emitter.prependListener('event', handlerC); // added to front — fires before A and B

// Execution order: C → A → B
```

## Debugging listener leaks

```javascript
// Find emitters with too many listeners
const { EventEmitter } = require('node:events');
EventEmitter.prototype.emit = (function(original) {
  return function(event) {
    const count = this.listenerCount(event);
    if (count > 20) {
      console.warn(`High listener count: ${count} on event '${event}'`);
      console.trace();
    }
    return original.apply(this, arguments);
  };
})(EventEmitter.prototype.emit);
```

## References

- https://nodejs.org/api/events.html
