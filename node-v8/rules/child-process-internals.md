---
name: child-process-internals
description: spawn, fork, IPC, handle passing, process pools
metadata:
  tags: child-process, ipc, spawn, fork, scaling
---

# Node.js Child Process Internals

## spawn vs. fork vs. exec

| Method | Use case | Shell |
|--------|----------|-------|
| `spawn` | Stream output, long-running | No |
| `fork` | Node.js worker with IPC | No |
| `exec` | Short command, buffer output | Yes |
| `execFile` | Short command, buffer output | No |

Prefer `execFile` over `exec` when possible (no shell = less overhead and injection risk).

## fork IPC

```javascript
// parent.js
const { fork } = require('node:child_process');
const child = fork('./worker.js');

child.send({ type: 'job', data: payload });
child.on('message', (result) => console.log('Result:', result));
child.on('exit', (code) => console.log('Exit:', code));

// worker.js
process.on('message', (msg) => {
  if (msg.type === 'job') {
    const result = doWork(msg.data);
    process.send({ result });
  }
});
```

## Handle passing (load balancing)

```javascript
// Parent: pass TCP server to children
const server = net.createServer();
server.listen(8000);
const child = fork('worker.js');
child.send({ type: 'server' }, server);

// worker.js
process.on('message', (msg, handle) => {
  if (msg.type === 'server') {
    handle.on('connection', (socket) => {
      // handle connection in this process
    });
  }
});
```

## Process pool

```javascript
class ProcessPool {
  constructor(script, size = 4) {
    this.free = [];
    this.queue = [];
    for (let i = 0; i < size; i++) {
      const w = fork(script);
      w.on('message', result => {
        const { resolve } = w.currentTask;
        w.currentTask = null;
        resolve(result);
        this.free.push(w);
        this._dispatch();
      });
      this.free.push(w);
    }
  }
  run(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this._dispatch();
    });
  }
  _dispatch() {
    if (!this.queue.length || !this.free.length) return;
    const w = this.free.pop();
    const { task, resolve, reject } = this.queue.shift();
    w.currentTask = { resolve, reject };
    w.send(task);
  }
  destroy() { this.free.forEach(w => w.kill()); }
}
```

## Detached daemon process

```javascript
const child = spawn('long-running', [], {
  detached: true,
  stdio: 'ignore',
});
child.unref(); // parent can exit without waiting
```

## References

- https://nodejs.org/api/child_process.html
