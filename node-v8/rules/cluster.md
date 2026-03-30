---
name: cluster
description: Cluster module - multi-core scaling, load balancing, graceful restart
metadata:
  tags: cluster, scaling, multi-core, load-balancing, ipc
---

# Node.js Cluster Module

Cluster spawns multiple Node.js processes (workers) that share the same server port,
utilizing all CPU cores. Workers are separate OS processes — no shared memory.

## Cluster vs. worker_threads

| | `cluster` | `worker_threads` |
|-|-----------|-----------------|
| Isolation | Full process isolation | Shared process |
| Memory | No shared memory (IPC only) | SharedArrayBuffer |
| Crash impact | Only that worker dies | Can crash whole process |
| Use case | HTTP server scaling | CPU-heavy parallelism |
| Overhead | Higher (fork) | Lower |

Use `cluster` when you want process isolation and crash resilience.
Use `worker_threads` when you need shared memory or low-overhead parallelism.

## Basic pattern

```javascript
const cluster = require('node:cluster');
const http = require('node:http');
const { availableParallelism } = require('node:os');
const process = require('node:process');

if (cluster.isPrimary) {
  const numCPUs = availableParallelism();
  console.log(`Primary ${process.pid} running, forking ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork(); // auto-restart
  });

} else {
  http.createServer((req, res) => {
    res.end(`Handled by worker ${process.pid}\n`);
  }).listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```

## Load balancing

**Default (all platforms except Windows):** round-robin — primary accepts connections,
distributes to workers evenly.

**Windows:** workers compete for connections — can be unbalanced (>70% to one worker
observed in production). Use a reverse proxy (nginx) for even distribution on Windows.

```javascript
// Force round-robin on all platforms
cluster.schedulingPolicy = cluster.SCHED_RR;
// or: NODE_CLUSTER_SCHED_POLICY=rr node app.js
```

## Graceful restart (zero-downtime deploy)

```javascript
if (cluster.isPrimary) {
  const workers = new Set();

  function spawnWorker() {
    const w = cluster.fork();
    workers.add(w);
    w.on('exit', () => workers.delete(w));
  }

  for (let i = 0; i < availableParallelism(); i++) spawnWorker();

  // On SIGUSR2: rolling restart
  process.on('SIGUSR2', () => {
    const current = [...workers];
    let i = 0;

    function restartNext() {
      if (i >= current.length) return;
      const old = current[i++];
      const fresh = cluster.fork();
      fresh.on('listening', () => {
        old.disconnect(); // stop accepting new connections
        restartNext();
      });
    }

    restartNext();
  });
}
```

## IPC between primary and workers

```javascript
// Primary → specific worker
worker.send({ type: 'config', data: newConfig });

// Worker → primary
process.send({ type: 'stats', requests: counter });

// Primary listens
cluster.on('message', (worker, msg) => {
  if (msg.type === 'stats') console.log(`Worker ${worker.id}:`, msg.requests);
});

// Worker listens
process.on('message', (msg) => {
  if (msg.type === 'config') applyConfig(msg.data);
});
```

## Shared state problem

Workers are separate processes — they cannot share in-memory state directly.

```javascript
// BAD: each worker has its own counter (won't aggregate correctly)
let requests = 0;
http.createServer((req, res) => {
  requests++; // invisible to other workers
  res.end();
});

// GOOD options:
// 1. Report to primary via IPC and aggregate there
// 2. Use external store (Redis, Memcached)
// 3. Use worker_threads with SharedArrayBuffer instead
```

## Graceful shutdown

```javascript
// Worker: finish in-flight requests before dying
process.on('SIGTERM', () => {
  server.close(() => {
    process.exit(0);
  });
});

// Primary: disconnect workers, wait for exit
process.on('SIGTERM', () => {
  for (const worker of Object.values(cluster.workers)) {
    worker.disconnect();
  }
});
```

## Worker lifecycle events

```javascript
cluster.on('fork',       (worker) => { /* worker forked */ });
cluster.on('online',     (worker) => { /* worker running */ });
cluster.on('listening',  (worker, addr) => { /* worker's server listening */ });
cluster.on('disconnect', (worker) => { /* worker disconnected from IPC */ });
cluster.on('exit',       (worker, code, signal) => { /* worker died */ });
```

## When NOT to use cluster

- CPU-bound work that doesn't need network — use `worker_threads`
- Shared in-memory caches — processes can't share (use Redis or `worker_threads`)
- Development — single process is simpler to debug
- Already behind a load balancer with multiple containers — redundant

## Production alternatives

- **PM2**: `pm2 start app.js -i max` — manages cluster automatically, restarts on crash,
  built-in monitoring
- **Multiple containers** behind a reverse proxy — better isolation, easier deployment

## References

- https://nodejs.org/api/cluster.html
