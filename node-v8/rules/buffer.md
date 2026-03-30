---
name: buffer
description: Buffer internals - allocation, encoding, pooling, performance patterns
metadata:
  tags: buffer, binary, encoding, performance, memory
---

# Node.js Buffer

Buffers represent fixed-length raw binary data. They are `Uint8Array` instances backed by
memory outside the V8 heap (tracked via external memory). Understanding Buffer allocation
and encoding is critical for performance in I/O-heavy applications.

## Allocation methods — choose carefully

```javascript
// SAFE: zero-filled, no data leakage
Buffer.alloc(size);
Buffer.alloc(size, fill);       // pre-fill with a value
Buffer.alloc(size, fill, encoding);

// FAST but UNSAFE: uninitialized — may contain old memory data
// Only use when you'll immediately overwrite all bytes
Buffer.allocUnsafe(size);

// Creates from existing data (always safe)
Buffer.from(array);             // copy from Uint8Array / number[]
Buffer.from(arrayBuffer, byteOffset, length); // view into ArrayBuffer
Buffer.from(string, encoding);  // encode string → Buffer
Buffer.from(buffer);            // copy another Buffer
```

## The internal pool — how allocUnsafe works

`Buffer.allocUnsafe()` for buffers < 4 KB does NOT malloc each time.
It allocates from a shared 8 KB pre-allocated pool (using `Buffer.allocUnsafeSlow()`
for the pool itself). This makes small allocations very fast but means the returned
buffer contains previous memory content — never return it to users without overwriting.

```javascript
// Bypasses pool — always new allocation (use for large buffers or when pooling matters)
Buffer.allocUnsafeSlow(size);
```

## Encodings

```javascript
buf.toString('utf8');    // default — multi-byte Unicode
buf.toString('ascii');   // 7-bit ASCII only, faster but lossy for non-ASCII
buf.toString('base64');  // for binary-safe transport (33% larger)
buf.toString('base64url'); // URL-safe base64 (no +, /, =)
buf.toString('hex');     // 2 hex chars per byte
buf.toString('binary');  // latin1 alias — 1 byte per char
buf.toString('latin1');  // ISO-8859-1
buf.toString('ucs2');    // UTF-16 little-endian (Windows paths)

Buffer.byteLength('hello', 'utf8'); // exact byte count before allocation
```

## Reading and writing binary data

```javascript
const buf = Buffer.alloc(16);

// Write
buf.writeUInt32BE(0xDEADBEEF, 0); // big-endian 32-bit at offset 0
buf.writeUInt32LE(12345, 4);       // little-endian 32-bit at offset 4
buf.writeDoubleBE(Math.PI, 8);     // 64-bit float at offset 8

// Read
buf.readUInt32BE(0);   // → 0xDEADBEEF
buf.readDoubleBE(8);   // → 3.141592653589793

// Methods: readInt8/UInt8, readInt16BE/LE, readInt32BE/LE,
//          readBigInt64BE/LE, readFloatBE/LE, readDoubleBE/LE
```

## Comparison and search

```javascript
buf1.equals(buf2);               // constant-time comparison (NOT timing-safe for secrets!)
buf1.compare(buf2);              // -1, 0, 1 (for sorting)
buf.indexOf(value, offset);      // find sub-buffer, byte, or string
buf.includes(value);             // boolean
```

For secrets: `crypto.timingSafeEqual(buf1, buf2)`.

## Slicing — beware of shared memory

```javascript
const original = Buffer.alloc(10, 0xFF);

// slice() and subarray() share memory with original!
const shared = original.slice(2, 6);
shared[0] = 0x00;
console.log(original[2]); // 0x00 — original mutated!

// SAFE: explicit copy
const copy = Buffer.from(original.slice(2, 6));
```

## Concatenation — pre-allocate vs. concat

```javascript
// BAD: concat in a loop creates many intermediate buffers
let result = Buffer.alloc(0);
for (const chunk of chunks) {
  result = Buffer.concat([result, chunk]); // O(n²) allocations
}

// GOOD: collect then concat once
const result = Buffer.concat(chunks); // single allocation

// BETTER: pre-allocate if total size is known
const totalSize = chunks.reduce((sum, c) => sum + c.length, 0);
const result = Buffer.allocUnsafe(totalSize);
let offset = 0;
for (const chunk of chunks) {
  chunk.copy(result, offset);
  offset += chunk.length;
}
```

## Buffer pooling pattern (high-throughput I/O)

```javascript
class BufferPool {
  constructor(bufferSize = 64 * 1024, poolSize = 32) {
    this.bufferSize = bufferSize;
    this.pool = Array.from({ length: poolSize }, () => Buffer.allocUnsafe(bufferSize));
  }

  acquire() {
    return this.pool.length > 0
      ? this.pool.pop()
      : Buffer.allocUnsafe(this.bufferSize);
  }

  release(buf) {
    if (buf.length === this.bufferSize) {
      buf.fill(0); // zero before returning to pool
      this.pool.push(buf);
    }
  }
}
```

## Buffer and streams — optimal chunk sizes

```javascript
// highWaterMark controls chunk size in streams
const stream = fs.createReadStream('file.bin', {
  highWaterMark: 64 * 1024, // 64 KB chunks (default: 16 KB)
});

// For network sockets: 16–64 KB is generally optimal
// For local disk: 64–256 KB reduces syscall overhead
```

## Converting to/from ArrayBuffer

```javascript
// Buffer → ArrayBuffer (zero-copy if possible)
const ab = buf.buffer.slice(buf.byteOffset, buf.byteOffset + buf.byteLength);

// ArrayBuffer → Buffer (zero-copy view)
const buf = Buffer.from(arrayBuffer);

// Buffer IS a Uint8Array
buf instanceof Uint8Array; // true
```

## Common mistakes

```javascript
// BAD: String concatenation with binary data
const str = binaryBuf.toString(); // defaults to utf8 — corrupts binary data

// BAD: Large base64 blobs in JSON API responses
res.json({ image: imageBuffer.toString('base64') }); // 33% overhead + JSON overhead

// BAD: Not checking bounds before read/write
buf.writeUInt32BE(value, offset); // throws if offset + 4 > buf.length

// BAD: Forgetting that slice shares memory
function getHeader(buf) {
  return buf.slice(0, 4); // caller may mutate the original buffer!
}
function getHeaderSafe(buf) {
  return Buffer.from(buf.slice(0, 4)); // copy
}
```

## Memory accounting

Buffers live outside V8 heap — visible in `process.memoryUsage().external`
and `process.memoryUsage().arrayBuffers`.

```javascript
const { external, arrayBuffers } = process.memoryUsage();
// If external grows unboundedly → Buffer leak
// Compare with heapUsed to understand where memory is going
```

## References

- https://nodejs.org/api/buffer.html
