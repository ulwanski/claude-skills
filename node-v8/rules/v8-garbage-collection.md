---
name: v8-garbage-collection
description: V8 GC internals - Scavenger, Mark-Sweep, Mark-Compact, generational GC
metadata:
  tags: v8, gc, garbage-collection, memory, performance
---

# V8 Garbage Collection

V8 uses a generational GC. Understanding it helps write performant apps and debug memory issues.

## Heap Spaces

- **New Space** (young gen) — Scavenger, semi-space copy, ~1–8 MB
- **Old Space** (old gen) — Mark-Sweep / Mark-Compact, ~1.4 GB default
- **Large Object Space** — objects > 512 KB, never moved
- **Code Space** — JIT-compiled machine code

## Configure heap

```bash
node --max-old-space-size=4096 app.js   # 4 GB old space
node --max-semi-space-size=64 app.js    # larger young gen
```

## Scavenger (minor GC — fast)

Semi-space copying: live objects copied to "to" space; dead ones abandoned.
Objects surviving two collections are promoted to old space.

**Implication**: many short-lived objects are cheap. Avoid promoting unnecessary
objects to old space by not storing them in long-lived data structures.

## Mark-Sweep (major GC — slower)

1. **Mark** — traverse from roots, mark reachable objects
2. **Sweep** — free unmarked objects

V8 uses incremental marking to reduce pause time (interleaved with JS execution).

## Mark-Compact

When fragmentation is high, live objects are compacted (moved together).
More expensive — V8 avoids it when possible.

## Heap statistics

```javascript
const v8 = require('node:v8');
const s = v8.getHeapStatistics();
console.log({
  used: (s.used_heap_size / 1e6).toFixed(1) + ' MB',
  total: (s.total_heap_size / 1e6).toFixed(1) + ' MB',
  limit: (s.heap_size_limit / 1e6).toFixed(1) + ' MB',
  external: (s.external_memory / 1e6).toFixed(1) + ' MB',
});
```

## Trace GC

```bash
node --trace-gc app.js
# Output: Scavenge 4.2 → 3.8 MB 1.2ms / Mark-sweep 15 → 12 MB 50ms
```

## Common pitfalls

### Closure retaining large data

```javascript
// BAD: closure retains largeData even if only summary is used
function createProcessor(largeData) {
  const summary = processData(largeData);
  return () => summary; // largeData cannot be collected!
}

// GOOD: break the closure chain
function createProcessor(largeData) {
  const summary = processData(largeData);
  largeData = null; // explicit release
  return () => summary;
}
```

### Timer retaining objects

```javascript
class Service {
  start() {
    this.timer = setInterval(() => this.doWork(), 1000);
  }
  stop() {
    clearInterval(this.timer); // MUST stop or Service leaks
    this.timer = null;
  }
}
```

## WeakRef / FinalizationRegistry

```javascript
const cache = new Map();
const registry = new FinalizationRegistry((key) => cache.delete(key));

function store(key, obj) {
  cache.set(key, new WeakRef(obj));
  registry.register(obj, key);
}
function retrieve(key) {
  return cache.get(key)?.deref();
}
```

## References

- https://v8.dev/blog/trash-talk
- node --v8-options | grep gc
