---
name: libuv-event-loop
description: libuv event loop phases, timers, I/O, idle, check, close
metadata:
  tags: libuv, event-loop, async, timers, io, phases
---

# libuv Event Loop

The event loop is the heart of Node.js's asynchronous model. Understanding its phases is essential for debugging timing issues and optimizing performance.

## Event Loop Architecture

```
   ┌───────────────────────────┐
┌─>│           timers          │ <- setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │ <- I/O callbacks deferred from previous loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │ <- internal use only
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │ <- setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │ <- socket.on('close', ...)
   └───────────────────────────┘
```

## Microtasks and nextTick

`process.nextTick()` and Promise callbacks execute between every phase:

```
       ┌──────────────────┐
       │ Current Phase    │
       └────────┬─────────┘
                ▼
    ┌───────────────────────┐
    │   nextTick queue      │ <- process.nextTick()
    └───────────┬───────────┘
                ▼
    ┌───────────────────────┐
    │   microtask queue     │ <- Promise.resolve().then()
    └───────────┬───────────┘
                ▼
       ┌────────────────┐
       │  Next Phase    │
       └────────────────┘
```

## queueMicrotask() vs. process.nextTick()

Both defer execution, but they use different queues and have different ordering depending
on the module system.

### Execution order — CJS

In CommonJS, `nextTick` fires before microtasks (including `queueMicrotask` and Promises):

```javascript
// CJS module
Promise.resolve().then(() => console.log('promise'));
queueMicrotask(() => console.log('microtask'));
process.nextTick(() => console.log('nextTick'));
// Output:
// nextTick
// promise
// microtask
```

### Execution order — ESM

In ES Modules, the file is itself processed inside the microtask queue, so
`queueMicrotask()` fires **before** `process.nextTick()`:

```javascript
// ESM module (.mjs or "type": "module")
import { nextTick } from 'node:process';
Promise.resolve().then(() => console.log('promise'));
queueMicrotask(() => console.log('microtask'));
nextTick(() => console.log('nextTick'));
// Output:
// promise
// microtask
// nextTick   ← reversed compared to CJS!
```

### When to use which

| | `process.nextTick()` | `queueMicrotask()` |
|-|----------------------|--------------------|
| Queue | nextTick queue | Microtask queue (same as Promise) |
| Portable across CJS/ESM | ⚠️ order differs | ✅ consistent with Promises |
| Pass arguments | ✅ `nextTick(fn, a, b)` | ❌ use closure/bind |
| Error handling | `process.on('uncaughtException')` | same |
| Recommendation | When you need nextTick-specific behavior | All other cases |

**Rule of thumb:** prefer `queueMicrotask()` for new code — it's a web-standard API,
consistent with Promise ordering, and works predictably across CJS and ESM.

```javascript
// GOOD: portable, consistent with Promise semantics
queueMicrotask(() => processNextItem());

// Use nextTick only when you need argument passing without closure:
process.nextTick(doWork, arg1, arg2);
// equivalent to (but slightly more efficient than):
queueMicrotask(() => doWork(arg1, arg2));
```

### Starvation applies to both

Just like `process.nextTick()`, recursive `queueMicrotask()` will starve I/O:

```javascript
// BAD: starves the event loop
function loop() { queueMicrotask(loop); }
loop();

// GOOD: use setImmediate for recursion that should yield to I/O
function loop() { setImmediate(loop); }
```

## setTimeout vs setImmediate

Within an I/O callback, `setImmediate` always fires before `setTimeout(fn, 0)`:

```javascript
const fs = require('node:fs');
fs.readFile('/etc/passwd', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Always: immediate, timeout
```

## nextTick Starvation

```javascript
// BAD: Recursive nextTick starves I/O
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
}
// I/O callbacks will NEVER execute!

// GOOD: Use setImmediate for recursion
function recursiveImmediate() {
  setImmediate(recursiveImmediate);
}
```

## Measuring Event Loop Lag

```javascript
const { monitorEventLoopDelay } = require('node:perf_hooks');
const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();
setInterval(() => {
  console.log({
    min: histogram.min / 1e6 + 'ms',
    p99: histogram.percentile(99) / 1e6 + 'ms',
    max: histogram.max / 1e6 + 'ms',
  });
  histogram.reset();
}, 5000);
```

## Common Issues

### Blocking the Event Loop

```javascript
// BAD: Sync I/O
const data = fs.readFileSync('large-file.txt');

// BAD: CPU-intensive sync computation
function fibonacci(n) { return n <= 1 ? n : fibonacci(n-1) + fibonacci(n-2); }
fibonacci(45); // Blocks for seconds!

// GOOD: Move to worker thread
const { Worker } = require('node:worker_threads');
```

## libuv Event Loop in C (simplified)

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  while (uv__loop_alive(loop)) {
    uv__update_time(loop);
    uv__run_timers(loop);
    uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);
    timeout = uv_backend_timeout(loop);
    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);
  }
}
```

## References

- libuv: http://docs.libuv.org/
- Node.js Event Loop guide: https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/
