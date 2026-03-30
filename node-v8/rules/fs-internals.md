---
name: fs-internals
description: libuv fs operations, sync vs async, file descriptor management
metadata:
  tags: fs, filesystem, async, sync, internals
---

# Node.js File System Internals

## All async fs operations use the thread pool

```
fs.readFile() → C++ FSReqCallback → libuv thread pool → OS read() → callback
```

Exceptions: `fs.watch()` (uses OS inotify/FSEvents/ReadDirectoryChangesW — no thread pool).

## Sync operations block the event loop entirely — avoid in servers

```javascript
// BAD in hot path
const data = fs.readFileSync('config.json');

// GOOD
const data = await fs.promises.readFile('config.json');
```

## File descriptor management

```javascript
// Always close file descriptors
const handle = await fs.promises.open('file.txt', 'r');
try {
  const buf = Buffer.alloc(1024);
  const { bytesRead } = await handle.read(buf, 0, 1024, 0);
  return buf.slice(0, bytesRead);
} finally {
  await handle.close(); // MUST close — even on error
}
```

## EMFILE (too many open files)

```javascript
// BAD: open all 10000 files at once
const contents = await Promise.all(files.map(f => fs.promises.readFile(f)));

// GOOD: limit concurrency
import pLimit from 'p-limit';
const limit = pLimit(100);
const contents = await Promise.all(files.map(f => limit(() => fs.promises.readFile(f))));
```

## Streaming large files

```javascript
// BAD: entire file in memory
const data = await fs.promises.readFile('huge.log'); // can be GBs

// GOOD: stream chunks
for await (const chunk of fs.createReadStream('huge.log')) {
  processChunk(chunk);
}
```

## Race conditions

```javascript
// BAD: check-then-use race
if (await fs.promises.stat(path).catch(() => null)) {
  const data = await fs.promises.readFile(path); // may still fail!
}

// GOOD: just try it
try {
  return await fs.promises.readFile(path);
} catch (err) {
  if (err.code === 'ENOENT') return null;
  throw err;
}
```

## Debugging

```bash
NODE_DEBUG=fs node app.js       # log all fs calls
strace -e trace=file node app.js  # Linux: syscall trace
```

## References

- https://nodejs.org/api/fs.html
