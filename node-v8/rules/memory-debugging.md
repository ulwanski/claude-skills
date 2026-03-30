---
name: memory-debugging
description: Heap snapshots, memory leak detection, debugging memory issues
metadata:
  tags: memory, debugging, heap-snapshots, memory-leaks, v8
---

# Memory Debugging

## process.memoryUsage()

```javascript
const u = process.memoryUsage();
// rss       - total process memory (including C++ heap)
// heapTotal - V8 heap capacity
// heapUsed  - V8 heap in use
// external  - C++ objects referenced by JS (Buffers, etc.)
// arrayBuffers - ArrayBuffer + SharedArrayBuffer allocations
```

## Taking heap snapshots

```javascript
const v8 = require('node:v8');
v8.writeHeapSnapshot('before.heapsnapshot');
// ... run suspect code ...
if (global.gc) global.gc();
v8.writeHeapSnapshot('after.heapsnapshot');
// Open both in Chrome DevTools → Memory → Comparison
```

```bash
# Take snapshot on signal without touching code
node --heapsnapshot-signal=SIGUSR2 app.js
# then: kill -USR2 $(pgrep node)
```

```bash
# Auto-snapshot near OOM (3 snapshots before crash)
node --heapsnapshot-near-heap-limit=3 app.js
```

## Reading snapshots

Key columns in Chrome DevTools:
- **Shallow size** — memory of the object itself
- **Retained size** — memory freed if this object is GC'd
- **Distance** — hops from GC root

Look for: objects with high retained size, growing delta between snapshots.

## Leak patterns

### Unbounded cache

```javascript
// BAD
const cache = new Map();
function get(key) {
  if (!cache.has(key)) cache.set(key, fetch(key));
  return cache.get(key);
}

// GOOD: LRU with TTL
const { LRUCache } = require('lru-cache');
const cache = new LRUCache({ max: 500, ttl: 5 * 60_000 });
```

### Event listener accumulation

```javascript
// BAD: never removed
emitter.on('data', handler);

// GOOD: return cleanup function
function subscribe(emitter, handler) {
  emitter.on('data', handler);
  return () => emitter.off('data', handler);
}
```

### Timer not cleared

```javascript
class Service {
  start() { this.timer = setInterval(() => this.work(), 1000); }
  stop()  { clearInterval(this.timer); this.timer = null; }
}
```

## Continuous leak monitoring

```javascript
const v8 = require('node:v8');
let baseline = null;

setInterval(() => {
  const used = v8.getHeapStatistics().used_heap_size;
  if (!baseline) { baseline = used; return; }
  const growthMb = (used - baseline) / 1e6;
  if (growthMb > 50) console.warn(`Heap grew ${growthMb.toFixed(1)} MB since start`);
}, 30_000);
```

## References

- https://developer.chrome.com/docs/devtools/memory-problems/
- https://v8.dev/blog/trash-talk
