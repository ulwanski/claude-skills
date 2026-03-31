---
name: dont-block-event-loop
description: Patterns that block the event loop - REDOS, JSON DoS, sync APIs, task partitioning
metadata:
  tags: event-loop, blocking, redos, performance, security, dos
---

# Don't Block the Event Loop (or the Worker Pool)

Node.js uses a small number of threads to handle many clients. Each callback on the Event
Loop and each task in the Worker Pool must complete quickly. If either is blocked, all
other clients wait.

Two threats: **performance** (throughput drops) and **security** (a malicious client can
craft input that blocks your thread — Denial of Service).

## Synchronous APIs that block the event loop

Never use these in a server request path:

```javascript
// Crypto — block main thread for seconds on slow hardware
crypto.randomBytesSync(256);
crypto.randomFillSync(buffer);
crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');

// Compression
zlib.deflateSync(buffer);
zlib.inflateSync(buffer);

// File system — ALL *Sync variants
fs.readFileSync(path);
fs.writeFileSync(path, data);
fs.statSync(path);

// Child process
child_process.spawnSync('cmd', args);
child_process.execSync('cmd');
child_process.execFileSync('cmd', args);
```

Use their async counterparts or offload to `worker_threads`.

## REDOS — regex that blocks the event loop

A vulnerable regex can take **exponential time** on adversarial input, making a single
request block the entire event loop for seconds or indefinitely.

**Vulnerability patterns:**
1. Nested quantifiers: `(a+)+`, `(a*)*`
2. OR with overlapping alternatives: `(a|a)+`
3. Backreferences: `(a.*) \1`

```javascript
// BAD: doubly-nested quantifier — vulnerable to REDOS
if (filePath.match(/(\/.+)+$/)) { ... }
// Input: '///.../\n' (100 slashes + newline) → blocks event loop indefinitely

// GOOD: use simple string methods when possible
if (filePath.startsWith('/') && !filePath.includes('\n')) { ... }

// GOOD: use indexed search instead of complex regex
const idx = str.indexOf(pattern); // always O(n), never exponential
```

**Detection tools:** `safe-regex` npm package, `rxxr2`.

**Alternative:** `node-re2` module uses Google's RE2 engine (linear time guarantee),
but it's not 100% compatible with V8's regex syntax.

## JSON DoS — large JSON blocks the event loop

`JSON.parse()` and `JSON.stringify()` are O(n) but for large n they can block for
hundreds of milliseconds. Never parse unbounded JSON from clients on the event loop.

```javascript
// BAD: client controls the size of the JSON body
app.post('/data', express.json(), (req, res) => {
  const result = process(req.body); // if body is huge, event loop blocked
});

// GOOD: enforce a size limit
app.use(express.json({ limit: '100kb' }));

// GOOD: for truly large JSON, use streaming JSON parser (e.g. JSONStream)
const JSONStream = require('JSONStream');
req.pipe(JSONStream.parse('items.*')).on('data', processItem);
```

## Task partitioning — breaking up CPU work

If you must do CPU work on the event loop, partition it across iterations so other
requests get turns between chunks:

```javascript
// BAD: O(n²) computation blocks event loop for all n² iterations
app.get('/report', (req, res) => {
  const n = parseInt(req.query.n);
  const results = [];
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      results.push(compute(i, j));
    }
  }
  res.json(results);
});

// BETTER: partition into async chunks
async function computePartitioned(n, chunkSize = 100) {
  const results = [];
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      results.push(compute(i, j));
    }
    // yield to event loop every chunkSize outer iterations
    if (i % chunkSize === 0) await new Promise(r => setImmediate(r));
  }
  return results;
}

// BEST: offload to worker_threads entirely
const { Worker } = require('node:worker_threads');
```

## Reasoning about callback complexity

Each callback should run in **constant time regardless of input size**. If your callback
is O(n) or worse, bound the input:

```javascript
// Enforce maximum input sizes
app.post('/search', (req, res) => {
  const query = req.body.q;
  if (typeof query !== 'string' || query.length > 200) {
    return res.status(400).json({ error: 'Query too long' });
  }
  // now safe to use in O(n) operation
});
```

## Worker Pool can also be blocked

The same rules apply to Worker Pool tasks. If a single `fs.readFile()` reads a 10 GB
file, it holds one thread pool thread for the entire duration — starving other I/O.

```javascript
// BAD: read entire large file (holds thread pool slot for whole duration)
const data = await fs.promises.readFile('huge.log');

// GOOD: stream in chunks (releases thread pool between reads)
for await (const chunk of fs.createReadStream('huge.log')) {
  processChunk(chunk);
}
```

## References

- https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS
