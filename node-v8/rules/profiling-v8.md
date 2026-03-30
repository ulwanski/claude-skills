---
name: profiling-v8
description: --prof, --trace-opt, --trace-deopt, flame graphs
metadata:
  tags: profiling, v8, performance, flame-graphs, cpu-profiling
---

# V8 Profiling

## Flame graph — quickest start

```bash
npx 0x app.js
# Opens HTML flame graph in browser
```

## --prof / --prof-process

```bash
node --prof app.js
node --prof-process isolate-*.log > profile.txt
```

In the output:
- `*funcName` = optimized by TurboFan
- `~funcName` = interpreted (not yet optimized)
- High ticks in `[GC]` = memory pressure

## Tracing optimization

```bash
node --trace-opt app.js 2>&1 | grep "marking\|compiling\|completed"
node --trace-deopt app.js 2>&1 | grep "DEOPT\|Reason"
```

## GC tracing

```bash
node --trace-gc app.js
# Output:
# Scavenge  4.2 → 3.8 MB  1.2ms
# Mark-sweep 15 → 12 MB  50ms (+ 10ms incremental)
```

```bash
node --trace-gc-verbose app.js  # more detail
```

## V8 Inspector CPU profile

```javascript
const inspector = require('node:inspector');
const fs = require('node:fs');
const session = new inspector.Session();
session.connect();

session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // run code to profile...
    setTimeout(() => {
      session.post('Profiler.stop', (err, { profile }) => {
        fs.writeFileSync('profile.cpuprofile', JSON.stringify(profile));
        // Open in Chrome DevTools → Performance
      });
    }, 10_000);
  });
});
```

## Heap space statistics

```javascript
const v8 = require('node:v8');
v8.getHeapSpaceStatistics().forEach(s => {
  console.log(s.space_name, (s.space_used_size / 1e6).toFixed(1) + ' MB used');
});
```

## Linux perf + FlameGraph

```bash
perf record -F 99 -g -- node app.js
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

## Clinic.js (production-grade)

```bash
npm install -g clinic
clinic flame -- node app.js      # CPU flame graph
clinic doctor -- node app.js     # event loop / I/O diagnosis
clinic bubbleprof -- node app.js # async operation visualization
```

## References

- https://v8.dev/docs/profile
- https://github.com/davidmarkclements/0x
