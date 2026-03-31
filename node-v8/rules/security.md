---
name: security
description: Node.js security best practices - DoS, timing attacks, prototype pollution, injection
metadata:
  tags: security, dos, timing-attacks, prototype-pollution, injection, http
---

# Node.js Security Best Practices

## HTTP server DoS (CWE-400)

### Always handle socket errors

```javascript
// BAD: unhandled socket error crashes the entire server
const server = net.createServer((socket) => {
  socket.write('hello\r\n');
  socket.pipe(socket);
});

// GOOD: handle errors at socket level
const server = net.createServer((socket) => {
  socket.on('error', (err) => {
    console.error('Socket error:', err.code);
  });
  socket.pipe(socket);
});
```

### Configure HTTP server timeouts — defense against Slowloris

```javascript
const server = http.createServer(handler);

server.headersTimeout  = 10_000;  // time to receive all headers
server.requestTimeout  = 30_000;  // time to receive full request body
server.timeout        = 120_000;  // idle socket timeout
server.keepAliveTimeout = 5_000;  // keep-alive idle timeout

// Limit concurrent connections
server.maxRequestsPerSocket = 100;
```

### Reverse proxy as first line of defense

Use nginx/HAProxy in front of Node.js to handle: connection limits, rate limiting,
IP blocking, TLS termination. Node.js alone is not designed to be internet-facing.

## Timing attacks (CWE-208)

Never use `===` to compare secrets — it short-circuits and leaks length/content
via response time differences.

```javascript
// BAD: timing-sensitive comparison — leaks secret via response time
if (providedToken === storedToken) { ... }

// GOOD: constant-time comparison
const crypto = require('node:crypto');
if (!crypto.timingSafeEqual(
  Buffer.from(providedToken),
  Buffer.from(storedToken)
)) {
  throw new Error('Invalid token');
}
```

This applies to: API keys, HMAC signatures, password reset tokens, CSRF tokens.

## Prototype pollution

Occurs when user-controlled input can set `__proto__`, `constructor`, or `prototype`
on an object, affecting all objects in the process.

```javascript
// ATTACK VECTOR
const userInput = JSON.parse('{"__proto__": {"isAdmin": true}}');
Object.assign({}, userInput); // pollutes Object.prototype!

// BAD: merging untrusted input into objects
function merge(target, source) {
  for (const key in source) {
    target[key] = source[key]; // key could be '__proto__'
  }
}

// GOOD: validate keys
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') continue;
    target[key] = source[key];
  }
}

// GOOD: use null-prototype objects for user data
const data = Object.create(null);

// GOOD: use Map instead of plain objects for user-controlled keys
const map = new Map(Object.entries(userInput));
```

## DNS rebinding — never run inspector in production (CWE-346)

The `--inspect` flag opens a WebSocket server. A malicious website can use DNS
rebinding to reach it, gaining full control of the Node.js process.

```bash
# NEVER in production:
node --inspect app.js
node --inspect-brk app.js

# If you must debug a production process, bind only to localhost and use SSH tunnel
node --inspect=127.0.0.1:9229 app.js
```

```javascript
// Disable SIGUSR1 inspector activation in production
process.on('SIGUSR1', () => {
  console.warn('Inspector activation attempted — ignored in production');
});
```

## HTTP request smuggling (CWE-444)

Occurs when front-end proxy and Node.js interpret HTTP/1.1 request boundaries
differently. Attacker can "smuggle" requests past the proxy.

```javascript
// NEVER use insecureHTTPParser
http.createServer({ insecureHTTPParser: true }, handler); // DANGEROUS

// MITIGATIONS:
// 1. Use HTTP/2 end-to-end (removes ambiguity)
// 2. Keep Node.js updated (parser issues get fixed)
// 3. Configure your reverse proxy to normalize ambiguous requests
```

## Avoiding eval and code injection

```javascript
// NEVER eval user input
eval(req.body.code);                    // arbitrary code execution
new Function(req.body.fn)();            // same as eval
vm.runInThisContext(req.body.script);   // same risk

// vm.runInNewContext is sandboxed but NOT secure for untrusted code
// A determined attacker can escape the sandbox
const vm = require('node:vm');
vm.runInNewContext(untrustedCode); // NOT a security sandbox

// GOOD: use worker_threads with limited permissions for untrusted code
// Or use a dedicated sandboxing solution (isolated-vm, etc.)
```

## Sensitive data exposure

```javascript
// BAD: stack traces in production responses
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack }); // leaks internals
});

// GOOD: generic error messages externally, full detail in logs
app.use((err, req, res, next) => {
  console.error(err); // full detail for operators
  res.status(500).json({ error: 'Internal server error' }); // generic for clients
});

// BAD: logging secrets
console.log('Config:', process.env); // logs all env vars including secrets

// GOOD: whitelist what you log
console.log('Port:', process.env.PORT);
```

## Path traversal

```javascript
const path = require('node:path');

// BAD: user controls file path directly
app.get('/file', (req, res) => {
  res.sendFile(req.query.path); // ../../etc/passwd works!
});

// GOOD: resolve and verify path stays within allowed directory
const BASE_DIR = path.resolve('./public');

app.get('/file', (req, res) => {
  const requested = path.resolve(BASE_DIR, req.query.filename);
  if (!requested.startsWith(BASE_DIR + path.sep)) {
    return res.status(403).send('Forbidden');
  }
  res.sendFile(requested);
});
```

## Dependency security

```bash
npm audit                    # check for known vulnerabilities
npm audit --audit-level high # fail CI on high/critical only
npm publish --dry-run        # review what files will be published

# Use lockfiles (package-lock.json or yarn.lock) — commit them
# Set exact versions for production dependencies
```

## Subprocess injection

```javascript
// BAD: user input in shell command
child_process.exec(`ls ${userInput}`); // shell injection!

// GOOD: use execFile (no shell) with explicit args array
child_process.execFile('ls', [userInput], callback);

// GOOD: use spawn with shell: false (default)
child_process.spawn('ls', [userInput], { shell: false });
```

## References

- https://nodejs.org/en/learn/getting-started/security-best-practices
- https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html
- https://github.com/nicolo-ribaudo/tc39-proposal-shadowrealm (prototype pollution mitigations)
