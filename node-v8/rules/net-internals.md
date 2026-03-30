---
name: net-internals
description: TCP/UDP implementation, socket handling, connection management
metadata:
  tags: net, tcp, udp, sockets, internals
---

# Node.js Net Internals

TCP/UDP in Node.js: JS layer → `TCPWrap`/`UDPWrap` C++ bindings → libuv → OS.
Network I/O does NOT use the thread pool — it uses epoll/kqueue/IOCP directly.

## Socket options

```javascript
server.on('connection', (socket) => {
  socket.setNoDelay(true);         // disable Nagle's algorithm (lower latency)
  socket.setKeepAlive(true, 60000); // detect dead connections
  socket.setTimeout(30000);         // idle timeout
});
```

## Connection tracking & graceful shutdown

```javascript
const connections = new Set();
server.on('connection', (socket) => {
  connections.add(socket);
  socket.on('close', () => connections.delete(socket));
});

async function shutdown() {
  server.close(); // stop accepting new connections
  for (const socket of connections) socket.end();
  await new Promise(resolve => setTimeout(resolve, 10_000));
  for (const socket of connections) socket.destroy();
  process.exit(0);
}
process.on('SIGTERM', shutdown);
```

## Common socket errors

```javascript
socket.on('error', (err) => {
  switch (err.code) {
    case 'ECONNREFUSED': // server not running or wrong port
    case 'ECONNRESET':   // peer closed connection unexpectedly
    case 'ETIMEDOUT':    // connection or idle timeout
    case 'EPIPE':        // writing to closed socket
      break;
  }
});
```

## HTTP agent — connection reuse

```javascript
const http = require('node:http');
const agent = new http.Agent({
  keepAlive: true,
  keepAliveMsecs: 60_000,
  maxSockets: 50,
  maxFreeSockets: 10,
});
// All requests through this agent reuse connections
```

## Debugging

```javascript
console.log({
  localAddress: socket.localAddress,
  localPort: socket.localPort,
  remoteAddress: socket.remoteAddress,
  remotePort: socket.remotePort,
  bytesRead: socket.bytesRead,
  bytesWritten: socket.bytesWritten,
  destroyed: socket.destroyed,
  readyState: socket.readyState,
});
```

```bash
strace -f -e trace=network node app.js   # Linux syscall trace
```

## References

- https://nodejs.org/api/net.html
