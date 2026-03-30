---
name: nodejs-core
description: >
  Deep Node.js internals expertise for application developers. Use this skill when working on
  performance bottlenecks, event loop blocking, memory leaks, async/await issues, large file
  or stream processing, scaling with clusters or worker threads, shared memory patterns,
  security sandboxing, V8 optimization and deoptimization, libuv thread pool starvation,
  or debugging native C++ addons. Trigger this skill whenever you encounter slow Node.js
  apps, mysterious async behavior, high memory usage, crashes, or need to reason about
  what happens below the JavaScript layer.
metadata:
  tags: nodejs, v8, libuv, performance, memory, streams, workers, async, internals, security
---

# Node.js Internals — Application Developer Guide

This skill provides deep knowledge of Node.js internals as they apply to building, debugging,
and scaling production applications. It covers what happens below the JavaScript layer —
the V8 engine, libuv event loop, thread pool, streams, worker threads, and native memory.

## When to apply this skill

- Event loop lag, timer drift, or inexplicably slow async operations
- Memory leaks or unexpected heap growth
- Thread pool starvation (too many concurrent `fs`, `dns.lookup`, `crypto`, or `zlib` ops)
- Designing high-throughput stream pipelines
- Scaling with `cluster`, `worker_threads`, or `SharedArrayBuffer`
- V8 deoptimization or JIT performance regressions
- Writing or debugging C++ addons (N-API / node-addon-api)
- Security: sandboxing, resource limits, understanding what Node.js enforces vs. what you must enforce
- Reasoning about the async/sync boundary and error propagation

## Rule files — read the relevant one before answering

### Event Loop & libuv

- [rules/libuv-event-loop.md](rules/libuv-event-loop.md)
  Event loop phases (timers → pending → idle → poll → check → close), microtask/nextTick
  ordering, `setImmediate` vs `setTimeout`, detecting and measuring event loop lag.

- [rules/libuv-thread-pool.md](rules/libuv-thread-pool.md)
  Which operations use the thread pool (`fs`, `dns.lookup`, `crypto.pbkdf2`, `zlib`),
  default pool size (4), `UV_THREADPOOL_SIZE`, detecting starvation, alternatives.

- [rules/libuv-async-io.md](rules/libuv-async-io.md)
  libuv handles vs. requests, epoll/kqueue/IOCP, TCP/UDP internals, poll handles,
  uv_async_t for cross-thread signalling, backpressure at the libuv layer.

### V8 Engine

- [rules/v8-garbage-collection.md](rules/v8-garbage-collection.md)
  New space / old space, Scavenger (minor GC), Mark-Sweep / Mark-Compact (major GC),
  write barriers, generational hypothesis, heap size configuration, detecting leaks.

- [rules/v8-hidden-classes.md](rules/v8-hidden-classes.md)
  Hidden classes (Maps), inline caching states (mono/poly/mega), elements kinds for arrays,
  property type stability, patterns that break IC and cause deoptimization.

- [rules/v8-jit-compilation.md](rules/v8-jit-compilation.md)
  Ignition bytecode, TurboFan optimizing compiler, type feedback, deoptimization reasons
  (`wrong map`, `not a Smi`, `out of bounds`), escape analysis, writing JIT-friendly code.

### Streams & I/O

- [rules/streams-internals.md](rules/streams-internals.md)
  `StreamBase` C++ class, Readable/Writable/Duplex/Transform internals, high water mark,
  backpressure mechanics, `pipeline`, zero-copy patterns.

- [rules/fs-internals.md](rules/fs-internals.md)
  Async vs. sync file operations, `FSReqCallback`, file descriptor management, `EMFILE`,
  streaming large files, watch vs. watchFile, race conditions.

- [rules/net-internals.md](rules/net-internals.md)
  `TCPWrap`/`UDPWrap` C++ bindings, connection handling, socket options (`TCP_NODELAY`,
  keep-alive), graceful shutdown, connection draining.

### Scaling & Parallelism

- [rules/cluster.md](rules/cluster.md)
  Multi-core scaling with separate processes, round-robin load balancing, IPC between primary
  and workers, graceful rolling restart, shared-state problem and alternatives (Redis, worker_threads).

- [rules/worker-threads-internals.md](rules/worker-threads-internals.md)
  Worker creation (separate V8 isolate + libuv loop), `MessageChannel` / structured clone,
  transfer lists (zero-copy), `SharedArrayBuffer`, `Atomics` (wait/notify, CAS),
  worker pool patterns, deadlock avoidance.

- [rules/child-process-internals.md](rules/child-process-internals.md)
  `spawn`/`fork`/`exec` flow, IPC channel setup, handle passing across processes,
  stdio configuration, process pools, graceful shutdown patterns.

### Binary Data & Events

- [rules/buffer.md](rules/buffer.md)
  `Buffer.alloc` vs `allocUnsafe`, internal pool mechanics, encodings, binary read/write,
  slicing (shared memory gotcha), efficient concatenation, pooling patterns, ArrayBuffer interop.

- [rules/events.md](rules/events.md)
  EventEmitter internals, the `error` event, `maxListeners` and leak warning, common listener
  leak patterns and fixes, async listeners / `captureRejections`, `events.once`, `events.on`
  async iterator.

### Context & Observability

- [rules/async-local-storage.md](rules/async-local-storage.md)
  `AsyncLocalStorage` for request-scoped context propagation (request ID, trace ID, auth context)
  without prop-drilling. `run()` / `getStore()` / `exit()`, context loss scenarios, `AsyncResource.bind`,
  `snapshot()`, memory leak pitfalls, singleton module pattern.

### Memory & Performance Debugging

- [rules/memory-debugging.md](rules/memory-debugging.md)
  Heap snapshots (`v8.writeHeapSnapshot`), `process.memoryUsage()`, leak patterns
  (unbounded caches, event listener leaks, closure retention, timer leaks),
  Valgrind / ASan, `WeakRef` / `FinalizationRegistry`.

- [rules/profiling-v8.md](rules/profiling-v8.md)
  `--prof` + `--prof-process`, `--trace-opt`, `--trace-deopt`, flame graphs with `0x`
  and `perf`, inline cache tracing, GC tracing flags, Clinic.js.

### Crypto & Security

- [rules/crypto-internals.md](rules/crypto-internals.md)
  Thread-pool vs. main-thread crypto ops, `pbkdf2`/`scrypt`/`randomBytes` async patterns,
  buffer reuse, timing-safe comparison, IV uniqueness, FIPS / OpenSSL providers.

### Native Addons (read only when working with C++)

- [rules/napi.md](rules/napi.md)
  N-API (C layer): ABI stability, status checking, async workers, ThreadSafeFunction,
  object wrapping, buffer handling.

- [rules/node-addon-api.md](rules/node-addon-api.md)
  node-addon-api (C++ wrapper): type conversions, `ObjectWrap`, `AsyncWorker`,
  `AsyncProgressWorker`, `TypedThreadSafeFunction`, `MemoryManagement::AdjustExternalMemory`.

- [rules/native-memory.md](rules/native-memory.md)
  RAII in addons, smart pointers, buffer ownership, external memory tracking,
  preventing leaks across async boundaries, Valgrind / ASan for C++ addon code.

- [rules/debugging-native.md](rules/debugging-native.md)
  GDB / LLDB setup, breakpoints, backtraces, core dumps, debugging segfaults and deadlocks,
  V8 GDB helpers, remote debugging.

---

## Quick-reference: diagnosing common problems

### Event loop blocked / high latency

```bash
node --trace-event-categories v8,node,node.async_hooks app.js
```

```javascript
// Measure lag at runtime
const { monitorEventLoopDelay } = require('node:perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log('p99 lag:', h.percentile(99) / 1e6, 'ms');
  h.reset();
}, 5000);
```

See [rules/libuv-event-loop.md](rules/libuv-event-loop.md).

### Thread pool starvation

```javascript
// Canary: dns.lookup latency reveals pool saturation
const dns = require('node:dns');
function pingPool() {
  const t = process.hrtime.bigint();
  dns.lookup('localhost', () => {
    const ms = Number(process.hrtime.bigint() - t) / 1e6;
    if (ms > 10) console.warn('Pool lag:', ms.toFixed(1), 'ms');
  });
}
setInterval(pingPool, 1000);
```

```bash
UV_THREADPOOL_SIZE=16 node app.js
```

See [rules/libuv-thread-pool.md](rules/libuv-thread-pool.md).

### Memory leak investigation

```bash
node --heapsnapshot-signal=SIGUSR2 app.js
# then: kill -USR2 <pid>  (take snapshot at runtime)
```

Compare two snapshots in Chrome DevTools → Memory → Comparison view.
See [rules/memory-debugging.md](rules/memory-debugging.md).

### V8 deoptimization

```bash
node --trace-opt --trace-deopt app.js 2>&1 | grep -E "DEOPT|Reason"
```

Identify the function and reason, then apply fixes from
[rules/v8-jit-compilation.md](rules/v8-jit-compilation.md) and
[rules/v8-hidden-classes.md](rules/v8-hidden-classes.md).

### CPU profile / flame graph

```bash
npx 0x app.js
```

or

```bash
node --prof app.js && node --prof-process isolate-*.log > profile.txt
```

See [rules/profiling-v8.md](rules/profiling-v8.md).

### Segfault in native addon

```bash
gdb --args node app.js
# (gdb) run  →  bt
```

Check `HandleScope`, `EscapableHandleScope`, and `uv_close()` sequencing.
See [rules/debugging-native.md](rules/debugging-native.md).

---

## Key principles

1. **Never block the event loop** — move CPU-bound work to `worker_threads`, I/O to libuv.
2. **Thread pool is shared and small** — default 4 threads; tune `UV_THREADPOOL_SIZE`, prefer `dns.resolve*()` over `dns.lookup()`.
3. **Type stability drives V8 performance** — consistent object shapes, no property additions after construction, avoid megamorphic call sites.
4. **Backpressure is a contract** — always check `stream.write()` return value; use `pipeline()`.
5. **Memory tracking is manual for native code** — call `AdjustExternalMemory` for large C++ allocations; use RAII and smart pointers.
6. **Security is layered** — Node.js does not sandbox JS by default; use `--max-old-space-size`, `worker_threads` resource limits, and OS-level controls for untrusted code.
