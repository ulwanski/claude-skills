---
name: debugging-native
description: GDB / LLDB for C++ addons and Node.js native crashes
metadata:
  tags: debugging, gdb, lldb, native-addons, segfault
---

# Debugging Native Code

## Build with debug symbols

```bash
# Node.js core
./configure --debug && make -j$(nproc)

# Addon
node-gyp configure --debug && node-gyp build --debug
```

## LLDB (macOS / Linux)

```bash
lldb -- node app.js
(lldb) run
# on crash:
(lldb) bt          # backtrace
(lldb) bt all      # all threads
(lldb) frame info  # current frame
(lldb) frame variable # local variables
```

## GDB (Linux)

```bash
gdb --args node app.js
(gdb) run
(gdb) bt
(gdb) info threads
(gdb) thread 2
(gdb) frame 3
(gdb) info locals
```

## Core dumps

```bash
ulimit -c unlimited
echo "/tmp/core.%e.%p" | sudo tee /proc/sys/kernel/core_pattern
# run until crash, then:
lldb node -c /tmp/core.node.12345
(lldb) bt all
```

## AddressSanitizer — best for catching memory bugs

```bash
./configure --enable-asan && make -j$(nproc)
ASAN_OPTIONS=detect_leaks=1:abort_on_error=1 node app.js
```

For addons:
```python
# binding.gyp
"cflags": ["-fsanitize=address", "-fno-omit-frame-pointer"],
"ldflags": ["-fsanitize=address"],
```

## Segfault diagnosis decision tree

1. Reproduce with `gdb --args node app.js` → `run` → `bt`
2. Does `bt` show a V8 handle scope issue? → Check `HandleScope` / `EscapableHandleScope`
3. Does it point to a libuv callback? → Inspect `uv_close()` sequencing
4. No C++ frame? → Type mismatch passed from JS to native binding

## Watchpoints

```lldb
(lldb) watchpoint set variable myVar    # break when myVar changes
```

## Sampling (no debugger needed)

```bash
# macOS
sample node 10 -file /tmp/sample.txt

# Linux perf
perf record -g -p $(pgrep node) -- sleep 30 && perf report
```

## References

- https://lldb.llvm.org/use/map.html
- https://v8.dev/docs/debug
