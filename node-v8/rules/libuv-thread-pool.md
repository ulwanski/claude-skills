---
name: libuv-thread-pool
description: libuv thread pool size, blocking operations, UV_THREADPOOL_SIZE
metadata:
  tags: libuv, thread-pool, async, blocking, uv-threadpool-size, performance
---

# libuv Thread Pool

libuv uses a thread pool for operations that can't be performed asynchronously at the OS level.

## What uses the thread pool

- **File system operations** — all `fs.*` except FSWatcher
- **DNS** — `dns.lookup()` (NOT `dns.resolve*()`)
- **Crypto** — `pbkdf2`, `scrypt`, `randomBytes`, `generateKeyPair`
- **Zlib** — compression/decompression
- **Custom C++ addons** — via `uv_queue_work`

```
Main Thread (Event Loop)
        ├──> Timer callbacks (no thread pool)
        ├──> Network I/O (epoll/kqueue/IOCP — no thread pool)
        └──> Thread Pool (4 threads by default)
             ├── Thread 1: fs.readFile()
             ├── Thread 2: dns.lookup()
             ├── Thread 3: crypto.pbkdf2()
             └── Thread 4: zlib.gzip()
```

## Default size: 4 threads

Only 4 blocking operations can run concurrently.

```bash
# Must be set before Node.js starts
UV_THREADPOOL_SIZE=16 node app.js

# Max is 1024
# Too late if set inside app:
process.env.UV_THREADPOOL_SIZE = 16; // NO EFFECT
```

## Detecting starvation

```javascript
// dns.lookup is a canary — if it's slow, pool is saturated
const dns = require('node:dns');
function measureThreadPoolLatency() {
  const start = process.hrtime.bigint();
  dns.lookup('localhost', (err) => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    if (ms > 10) console.warn(`Thread pool lag: ${ms.toFixed(2)}ms`);
  });
}
setInterval(measureThreadPoolLatency, 1000);
```

## DNS: lookup vs. resolve

```javascript
// BAD: dns.lookup uses thread pool
await dns.promises.lookup('example.com');

// GOOD: dns.resolve* uses c-ares (no thread pool)
await dns.promises.resolve4('example.com');
```

## Crypto: thread pool vs. main thread

```javascript
// Uses thread pool (async):
crypto.pbkdf2(password, salt, 100000, 64, 'sha512', callback);
crypto.randomBytes(256, callback);
crypto.scrypt(password, salt, keylen, callback);

// Runs on main thread (sync — avoid in hot paths):
crypto.createHash('sha256').update(data).digest();
crypto.createCipheriv(algorithm, key, iv);
```

## Sizing formula

```javascript
const os = require('node:os');
// I/O-heavy: threads can wait, more is fine
const recommended = Math.max(os.cpus().length * 2, 4);
```

## Best practices

1. Set `UV_THREADPOOL_SIZE` to at least `numCPUs * 2` for I/O-heavy apps
2. Use `dns.resolve*()` instead of `dns.lookup()` wherever possible
3. Cache DNS results to avoid repeated thread pool usage
4. Stream large files instead of `readFile` to hold the pool slot for less time
5. Move CPU-heavy work to `worker_threads` (bypasses pool entirely)

## References

- http://docs.libuv.org/en/v1.x/threadpool.html
