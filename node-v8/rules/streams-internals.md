---
name: streams-internals
description: How Node.js streams work — backpressure, pipeline, internals
metadata:
  tags: streams, readable, writable, backpressure, pipeline, transform
---

# Node.js Streams Internals

## .pipe() does NOT propagate errors — always use pipeline()

```javascript
// BAD: if gzip fails, the write stream is not destroyed — file descriptor leak
const inp = fs.createReadStream('file.mkv');
const gz  = zlib.createGzip();
const out = fs.createWriteStream('file.mkv.gz');
inp.pipe(gz).pipe(out); // error in gz → out stays open

// GOOD: pipeline() destroys all streams on error and calls callback when done
const { pipeline } = require('node:stream/promises');
await pipeline(
  fs.createReadStream('file.mkv'),
  zlib.createGzip(),
  fs.createWriteStream('file.mkv.gz')
);
```

## Stream types

- **Readable** — source of data (`fs.createReadStream`, `http.IncomingMessage`)
- **Writable** — sink of data (`fs.createWriteStream`, `http.ServerResponse`)
- **Duplex** — both (TCP socket)
- **Transform** — Duplex that modifies data (`zlib.createGzip`, custom parsers)

## High water mark (HWM)

Buffer threshold that controls backpressure. Default: 16 KB for byte streams, 16 objects for object mode.

```javascript
const stream = fs.createReadStream('file.txt', { highWaterMark: 64 * 1024 });
```

## Backpressure — the contract

```javascript
// ALWAYS check write() return value
async function writeWithBackpressure(writable, chunks) {
  for (const chunk of chunks) {
    if (!writable.write(chunk)) {
      await once(writable, 'drain'); // wait for buffer to drain
    }
  }
}
```

## pipeline — the safe way to connect streams

```javascript
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');
const zlib = require('node:zlib');

await pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('output.gz')
);
// Handles errors, backpressure, and cleanup automatically
```

## Async iteration

```javascript
const fs = require('node:fs');

async function processLines(filePath) {
  const stream = fs.createReadStream(filePath, { encoding: 'utf8' });
  for await (const chunk of stream) {
    process(chunk);
  }
}
```

## Custom Transform stream

```javascript
const { Transform } = require('node:stream');

class UpperCase extends Transform {
  _transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
}
```

## Debugging stream state

```javascript
// Readable internals
const rs = getReadableStream();
console.log({
  hwm: rs._readableState.highWaterMark,
  buffered: rs._readableState.length,
  flowing: rs._readableState.flowing,
  ended: rs._readableState.ended,
});

// Writable internals
const ws = getWritableStream();
console.log({
  hwm: ws._writableState.highWaterMark,
  buffered: ws._writableState.length,
  writing: ws._writableState.writing,
  finished: ws._writableState.finished,
});
```

## Common mistakes

```javascript
// BAD: ignoring backpressure
for (const chunk of bigData) {
  stream.write(chunk); // may buffer unboundedly
}

// BAD: data lost on error
stream.on('data', chunk => processAsync(chunk)); // not awaited
stream.on('end', () => { /* processing may be incomplete */ });

// GOOD: async iteration ensures ordering
for await (const chunk of stream) {
  await processAsync(chunk);
}
```

## References

- https://nodejs.org/api/stream.html
- https://nodejs.org/en/docs/guides/backpressuring-in-streams
