---
name: v8-hidden-classes
description: V8 hidden classes, inline caching, optimization patterns
metadata:
  tags: v8, hidden-classes, inline-caching, optimization, performance
---

# V8 Hidden Classes (Maps)

V8 attaches a hidden class (Map) to each object describing its structure.
Property accesses are optimized by caching the Map and the property offset.

## Inline Cache States

1. **Monomorphic** — one Map seen → fastest (direct offset lookup)
2. **Polymorphic** — 2–4 Maps → still fast (small dispatch table)
3. **Megamorphic** — >4 Maps → falls back to dictionary lookup

## Rules for keeping ICs monomorphic

### Always add properties in the same order

```javascript
// BAD: different order = different Map
function Point(x, y) {
  if (x < 0) { this.x = x; this.y = y; }
  else        { this.y = y; this.x = x; } // ← reversed!
}

// GOOD
class Point {
  constructor(x, y) { this.x = x; this.y = y; }
}
```

### Never add properties after construction

```javascript
// BAD
const obj = { x: 1 };
if (cond) obj.z = 3; // creates a new Map

// GOOD
const obj = { x: 1, z: cond ? 3 : undefined };
```

### Never delete properties

```javascript
// BAD: delete may push object to slow (dictionary) mode
delete obj.y;

// GOOD
obj.y = undefined;
```

### Keep property types stable

```javascript
// BAD: type change creates new Map
obj.value = 42;
obj.value = "string"; // Map transition!
```

## Elements kinds (arrays)

```
PACKED_SMI_ELEMENTS     [1, 2, 3]             ← fastest
PACKED_DOUBLE_ELEMENTS  [1.1, 2.2]
PACKED_ELEMENTS         [1, "two", {}]
HOLEY_ELEMENTS          [1, , 3]              ← slowest
```

Transitions only go *down*; they never reverse. Avoid holes and mixed types.

```javascript
// BAD: creates HOLEY array
const arr = new Array(1000);
arr[0] = 1; // HOLEY_SMI

// GOOD
const arr = new Array(1000).fill(0); // PACKED_SMI
```

## Debugging

```bash
node --trace-maps app.js          # log Map transitions
node --trace-opt app.js           # log what gets optimized
node --trace-deopt app.js         # log deoptimizations + reasons
```

## References

- https://v8.dev/blog/fast-properties
- https://v8.dev/blog/elements-kinds
