---
name: libuv-async-io
description: libuv async I/O patterns, handles, requests
metadata:
  tags: libuv, async-io, handles, requests, epoll, kqueue, iocp
---

# libuv Async I/O

libuv uses platform-specific mechanisms: epoll (Linux), kqueue (macOS/BSD), IOCP (Windows).

## Handles vs. Requests

- **Handles** — long-lived resources (TCP socket, timer, file watcher)
- **Requests** — short-lived operations (uv_write_t, uv_fs_t, uv_work_t)

## Stream Backpressure

```javascript
const socket = net.connect(80, 'example.com');

socket.on('data', (chunk) => {
  socket.pause();          // → uv_read_stop()
  processChunk(chunk, () => {
    socket.resume();       // → uv_read_start()
  });
});
```

## Write Backpressure

```javascript
const canContinue = socket.write(data);
if (!canContinue) {
  socket.once('drain', () => { /* write more */ });
}
```

## uv_async_t — cross-thread signalling

Used by worker_threads and N-API async callbacks to wake the main loop:

```c
uv_async_t async;
uv_async_init(loop, &async, async_cb);  // main thread
// From any thread:
uv_async_send(&async);
```

## Active handles debugging

```javascript
// What's keeping the process alive?
process._getActiveHandles().forEach(h => console.log(h.constructor.name));
process._getActiveRequests().forEach(r => console.log(r.constructor.name));
```

## Graceful shutdown pattern

```javascript
const connections = new Set();
server.on('connection', (s) => {
  connections.add(s);
  s.on('close', () => connections.delete(s));
});

function shutdown() {
  server.close(() => process.exit(0));
  for (const s of connections) s.end();
  setTimeout(() => {
    for (const s of connections) s.destroy();
    process.exit(1);
  }, 10_000);
}
process.on('SIGTERM', shutdown);
```

## References

- http://docs.libuv.org/en/v1.x/design.html
