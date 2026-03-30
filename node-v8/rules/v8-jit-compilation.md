---
name: v8-jit-compilation
description: V8 JIT - TurboFan, optimization/deoptimization patterns
metadata:
  tags: v8, jit, turbofan, optimization, deoptimization, ignition
---

# V8 JIT Compilation

## Pipeline: Ignition → TurboFan

1. **Ignition** — bytecode interpreter; runs immediately, collects type feedback
2. **TurboFan** — optimizing JIT; compiles hot functions using collected feedback

```bash
node --print-bytecode --print-bytecode-filter=myFn app.js
node --trace-opt app.js    # what gets optimized
node --trace-deopt app.js  # what gets deoptimized + reason
```

## Deoptimization reasons

| Reason | Fix |
|--------|-----|
| `wrong map` | Consistent object shapes (see hidden-classes.md) |
| `not a Smi` | Keep integers in SMI range (-2^30 to 2^30-1) |
| `out of bounds` | Avoid sparse array access |
| `minus zero` | Avoid `-0` producing operations |
| `overflow` | Use BigInt for very large integers |

## Writing JIT-friendly code

### Type-stable functions

```javascript
// BAD: returns both number and string
function process(x) {
  return typeof x === 'number' ? x * 2 : x + x;
}

// GOOD: separate typed functions
function processNum(x) { return x * 2; }
function processStr(x) { return x + x; }
```

### Avoid megamorphic call sites

```javascript
// BAD: getLength called with Array, String, Object, Set
function getLength(obj) { return obj.length; }

// GOOD: separate per type, or use dedicated code paths
```

### Prefer rest params over arguments object

```javascript
// BAD: arguments object can prevent optimization
function sum() {
  let r = 0;
  for (let i = 0; i < arguments.length; i++) r += arguments[i];
  return r;
}

// GOOD
function sum(...args) {
  let r = 0;
  for (let i = 0; i < args.length; i++) r += args[i];
  return r;
}
```

### Avoid Object spread in hot loops

```javascript
// BAD: creates new object + new Map each iteration
for (const obj of items) {
  result = { ...result, ...obj };
}

// GOOD
for (const obj of items) {
  Object.assign(result, obj);
}
```

## Flame graph to find deopt loops

```bash
npx 0x app.js
# Look for wide bars with "deoptimize" or "unoptimized" in the label
```

## References

- https://v8.dev/docs/turbofan
- https://v8.dev/docs/ignition
