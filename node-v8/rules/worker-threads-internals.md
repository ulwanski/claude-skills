---
name: worker-threads-internals
description: SharedArrayBuffer, Atomics, MessageChannel internals
metadata:
  tags: worker-threads, shared-memory, atomics, parallelism
---

# Worker Threads Internals

Workers run in a separate V8 isolate with their own event loop — true parallelism.

## Creating a worker

```javascript
const { Worker, isMainThread, parentPort, workerData } = require('node:worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename, { workerData: { n: 42 } });
  worker.on('message', result => console.log('Result:', result));
  worker.on('error', err => console.error(err));
  worker.on('exit', code => console.log('Exit:', code));
} else {
  parentPort.postMessage(workerData.n * 2);
}
```

## MessageChannel — structured clone

```javascript
const { MessageChannel } = require('node:worker_threads');
const { port1, port2 } = new MessageChannel();

port1.on('message', msg => console.log('Got:', msg));
port2.postMessage({ hello: 'world' });
```

Supported types: primitives, Date, RegExp, Array, Map, Set, Buffer, TypedArray, ArrayBuffer, Error.
Not supported: functions, symbols, WeakMap/WeakSet.

## Transfer list — zero-copy

```javascript
const ab = new ArrayBuffer(10 * 1024 * 1024);
worker.postMessage({ buffer: ab }, [ab]);
// ab is now detached (unusable) in sender — ownership transferred
```

## SharedArrayBuffer — true shared memory

```javascript
// main thread
const sharedBuf = new SharedArrayBuffer(4096);
const sharedArr = new Int32Array(sharedBuf);
const worker = new Worker('./worker.js', { workerData: { sharedBuf } });
sharedArr[0] = 1; // both threads see this

// worker.js
const sharedArr = new Int32Array(workerData.sharedBuf);
console.log(sharedArr[0]); // 1
```

## Atomics — thread-safe operations

```javascript
Atomics.add(sharedArr, 0, 1);                         // atomic increment
Atomics.compareExchange(sharedArr, 0, expected, next); // CAS
Atomics.load(sharedArr, 0);                            // atomic read
Atomics.store(sharedArr, 0, 42);                       // atomic write
```

## Wait / Notify (futex)

```javascript
// Worker: block until value changes
Atomics.wait(sharedArr, 0, 0); // wait while arr[0] === 0

// Main: wake waiting workers
Atomics.store(sharedArr, 0, 1);
Atomics.notify(sharedArr, 0, 1); // wake 1 waiter

// NEVER call Atomics.wait() on the main thread — it blocks the event loop!
```

## Worker pool pattern

```javascript
class WorkerPool {
  constructor(script, size = require('node:os').cpus().length) {
    this.free = [];
    this.queue = [];
    for (let i = 0; i < size; i++) {
      const w = new Worker(script);
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
  run(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this._dispatch();
    });
  }
  _dispatch() {
    if (!this.queue.length || !this.free.length) return;
    const w = this.free.pop();
    const task = this.queue.shift();
    w.currentTask = task;
    w.postMessage(task.data);
  }
  terminate() {
    return Promise.all(this.free.map(w => w.terminate()));
  }
}
```

## Resource limits

```javascript
new Worker('./worker.js', {
  resourceLimits: {
    maxOldGenerationSizeMb: 128,
    maxYoungGenerationSizeMb: 32,
    stackSizeMb: 4,
  }
});
```

## Deadlock avoidance

- Never use `Atomics.wait()` on the main thread
- Always have a timeout: `Atomics.wait(arr, 0, 0, 5000)`
- Ensure notify is always called (use try/finally)

## References

- https://nodejs.org/api/worker_threads.html
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer
