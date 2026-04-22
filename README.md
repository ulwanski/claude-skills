# claude-skills

**What is a Claude Skill?**
A Claude Skill is a structured knowledge pack loaded into Claude's context when working on specific tasks. Instead of relying solely on general training data, Claude reads targeted rule files that contain precise, opinionated guidance - effectively giving Claude "expert mode" on a subject.

## nodejs-core

> Deep Node.js internals expertise for AI-assisted development.

A **Claude skill** that equips the AI with detailed, reference-grade knowledge of Node.js internals - from the V8 engine and libuv event loop, through streams, worker threads, and native addons, down to production hardening and security. Designed for developers building, debugging, and scaling production Node.js applications.

---

## Skill contents

```
nodejs-core/
├── SKILL.md                        # Skill entry point & quick-reference
└── rules/
    ├── libuv-event-loop.md         # Event loop phases, timers, microtasks
    ├── libuv-thread-pool.md        # Thread pool, UV_THREADPOOL_SIZE, starvation
    ├── libuv-async-io.md           # Handles, epoll/kqueue/IOCP, TCP/UDP internals
    ├── v8-garbage-collection.md    # New/old space, Scavenger, Mark-Compact, heap config
    ├── v8-hidden-classes.md        # Hidden classes, inline caches, elements kinds
    ├── v8-jit-compilation.md       # Ignition, TurboFan, deoptimization, JIT-friendly patterns
    ├── streams-internals.md        # Readable/Writable/Transform, HWM, backpressure, pipeline
    ├── fs-internals.md             # Async vs sync ops, FSReqCallback, EMFILE, large files
    ├── net-internals.md            # TCPWrap/UDPWrap, socket options, graceful shutdown
    ├── cluster.md                  # Multi-core scaling, IPC, rolling restarts, shared state
    ├── worker-threads-internals.md # V8 isolates, MessageChannel, SharedArrayBuffer, Atomics
    ├── child-process-internals.md  # spawn/fork/exec, IPC, handle passing, process pools
    ├── async-local-storage.md      # Request-scoped context, run/getStore, context loss, leaks
    ├── events.md                   # EventEmitter internals, error event, listener leaks
    ├── buffer.md                   # alloc vs allocUnsafe, pool mechanics, ArrayBuffer interop
    ├── memory-debugging.md         # Heap snapshots, process.memoryUsage, leak patterns
    ├── profiling-v8.md             # --prof, flame graphs with 0x, Clinic.js, GC tracing
    ├── crypto-internals.md         # Thread-pool vs main-thread crypto, timing-safe compare
    ├── security.md                 # HTTP DoS, prototype pollution, path traversal, DNS rebinding
    ├── dont-block-event-loop.md    # REDOS, dangerous sync APIs, JSON DoS, task partitioning
    ├── production.md               # NODE_ENV, 12-factor config, graceful shutdown, Docker/PID 1
    ├── napi.md                     # N-API C layer, async workers, ThreadSafeFunction
    ├── node-addon-api.md           # C++ wrapper, ObjectWrap, AsyncWorker, TypedThreadSafeFunction
    ├── native-memory.md            # RAII, smart pointers, external memory tracking, Valgrind
    └── debugging-native.md         # GDB/LLDB, backtraces, core dumps, V8 GDB helpers
```

**26 files · ~102 KB of reference material**

---

## Coverage areas

### Event Loop & libuv
Full documentation of the 6 event loop phases (timers → pending → idle → poll → check → close), microtask queue ordering (`process.nextTick` vs. `Promise` callbacks), `setImmediate` vs. `setTimeout` execution guarantees, and techniques for measuring event loop lag at runtime.

The thread pool section covers which standard library operations consume pool threads (`fs`, `dns.lookup`, `crypto.pbkdf2`, `zlib`), the default pool size of 4, tuning via `UV_THREADPOOL_SIZE`, starvation detection with a `dns.lookup` canary, and alternatives like `dns.resolve*()` or userland async libraries.

### V8 Engine
Three dedicated files covering the full JIT pipeline: Ignition bytecode interpreter, TurboFan optimizing compiler, type feedback vectors, deoptimization triggers (`wrong map`, `not a Smi`, `out of bounds`), and patterns for writing JIT-friendly code. The hidden class file covers object shape stability, inline cache states (monomorphic / polymorphic / megamorphic), elements kind transitions, and what breaks the IC. The GC file covers generational collection, write barriers, heap size flags, and heap snapshot comparison for leak detection.

### Streams & I/O
Internals of the `StreamBase` C++ class, how `highWaterMark` governs buffering, the mechanics of backpressure (checking `write()` return value, `drain` event, `pipeline()`), zero-copy patterns, and file system and network specifics including `EMFILE` avoidance and graceful TCP shutdown.

### Scaling & Parallelism
The cluster module for multi-process scaling (round-robin load balancing, IPC messaging, rolling restarts), worker threads for CPU-bound parallelism (separate V8 isolate + libuv loop per worker, `MessageChannel`, transfer lists for zero-copy, `SharedArrayBuffer` + `Atomics` for shared state), and child process patterns including handle passing across process boundaries.

### Security & Production
Practical attack surface coverage: HTTP DoS (server timeouts, `headersTimeout`, `requestTimeout`), REDOS and other event loop DoS vectors, prototype pollution detection and fixes, timing-safe comparison with `crypto.timingSafeEqual`, DNS rebinding, path traversal, and subprocess injection. The production file covers `NODE_ENV`, 12-factor environment config, startup validation, graceful shutdown sequences, `uncaughtException`/`unhandledRejection` handling, and running Node.js as PID 1 in Docker.

### Observability & Debugging
`AsyncLocalStorage` for request-scoped context propagation (trace IDs, auth context) without prop drilling, including context loss scenarios and memory leak pitfalls. Memory debugging with heap snapshots, `--heapsnapshot-signal`, and Chrome DevTools comparison view. CPU profiling with `--prof`, `node --prof-process`, `0x` flame graphs, and Clinic.js.

### Native Addons *(C++)*
N-API (C layer) and node-addon-api (C++ wrapper) - ABI stability, async workers, `ThreadSafeFunction`, `ObjectWrap`, `AsyncProgressWorker`, and external memory tracking via `AdjustExternalMemory`. Debugging native addons with GDB/LLDB, V8 GDB helpers, core dumps, and segfault analysis.

---

## Quick diagnostics

| Symptom | Tool | Rule file |
|---|---|---|
| High latency / event loop lag | `monitorEventLoopDelay`, `--trace-event-categories` | `libuv-event-loop.md` |
| Thread pool starvation | `dns.lookup` canary, `UV_THREADPOOL_SIZE` | `libuv-thread-pool.md` |
| Memory leak | `--heapsnapshot-signal=SIGUSR2`, heap snapshot diff | `memory-debugging.md` |
| V8 deoptimization | `--trace-opt --trace-deopt` | `v8-jit-compilation.md` |
| CPU bottleneck | `npx 0x`, `--prof` | `profiling-v8.md` |
| Segfault in addon | `gdb --args node`, `bt` | `debugging-native.md` |
| Backpressure / OOM on streams | Check `write()` return, use `pipeline()` | `streams-internals.md` |
| REDOS / event loop DoS | Audit regex, use `safe-regex` | `dont-block-event-loop.md` |

---

## Usage

### With Claude.ai (skills feature)

Upload the `nodejs-core/` directory as a skill in your Claude workspace. Claude will automatically read the relevant rule file before answering Node.js internals questions.

### With the Claude API

Pass the contents of `SKILL.md` (and the relevant `rules/*.md`) in the system prompt or as a document block before your user message:

```javascript
import Anthropic from '@anthropic-ai/sdk';
import fs from 'node:fs';

const client = new Anthropic();
const skill = fs.readFileSync('./nodejs-core/SKILL.md', 'utf8');
const rule  = fs.readFileSync('./nodejs-core/rules/libuv-event-loop.md', 'utf8');

const response = await client.messages.create({
  model: 'claude-opus-4-5',
  max_tokens: 2048,
  system: `${skill}\n\n---\n\n${rule}`,
  messages: [
    { role: 'user', content: 'Why does setImmediate fire before setTimeout(fn, 0) in some cases?' }
  ],
});

console.log(response.content[0].text);
```

---

## Design principles

The rule files follow a consistent format:

- **Concise frontmatter** - name, description, tags for quick lookup
- **Concrete code examples** - every pattern is illustrated with minimal, runnable snippets
- **Diagnostic commands** - bash one-liners for immediate use
- **Cross-references** - each file links to related rules when a topic spans multiple layers
- **No fluff** - written for senior developers who want precise answers, not introductory explanations

---

## Requirements

- Node.js ≥ 18 (some APIs covered require ≥ 20 / ≥ 22)
- Claude Sonnet 4 or Opus 4 recommended for best internals reasoning

---

## License

MIT
