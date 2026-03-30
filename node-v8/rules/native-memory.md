---
name: native-memory
description: Buffer handling, external memory, preventing leaks in C++ addons
metadata:
  tags: native-memory, buffers, memory-leaks, native-addons
---

# Native Memory Management in Addons

## Use RAII — prefer smart pointers over raw

```cpp
// BAD
uint8_t* buf = new uint8_t[1024];
// ... if exception thrown: leak!
delete[] buf;

// GOOD
std::unique_ptr<uint8_t[]> buf = std::make_unique<uint8_t[]>(1024);
// automatically freed on scope exit or exception
```

## Buffer ownership patterns

```cpp
// 1. Node.js allocates and owns
Napi::Buffer<uint8_t>::New(env, size);

// 2. Copy existing data (safe, original can be freed)
Napi::Buffer<uint8_t>::Copy(env, data.data(), data.size());

// 3. External buffer — you own the memory, provide finalizer
Napi::Buffer<uint8_t>::New(env, data, size,
  [](Napi::Env, uint8_t* d) { delete[] d; }
);
```

## Buffer lifetime — never store raw pointer

```cpp
// BAD: buffer may be GC'd while pointer is held
class Addon : public Napi::ObjectWrap<Addon> {
  uint8_t* ptr_; // DANGEROUS
  void Store(const Napi::CallbackInfo& info) {
    ptr_ = info[0].As<Napi::Buffer<uint8_t>>().Data();
  }
};

// GOOD: keep a reference to prevent GC
class Addon : public Napi::ObjectWrap<Addon> {
  Napi::Reference<Napi::Buffer<uint8_t>> ref_;
  void Store(const Napi::CallbackInfo& info) {
    ref_ = Napi::Persistent(info[0].As<Napi::Buffer<uint8_t>>());
  }
};
```

## Tell V8 about large native allocations

V8's GC doesn't know about malloc'd memory — if you don't report it, GC
won't feel pressure to collect and you'll OOM.

```cpp
// When allocating
Napi::MemoryManagement::AdjustExternalMemory(env, +sizeBytes);

// When freeing (in finalizer/destructor callback, not C++ destructor)
Napi::MemoryManagement::AdjustExternalMemory(env, -sizeBytes);
```

## ThreadSafeFunction — always Release

```cpp
class EventEmitter : public Napi::ObjectWrap<EventEmitter> {
  Napi::ThreadSafeFunction tsfn_;
  void Listen(const Napi::CallbackInfo& info) {
    tsfn_ = Napi::ThreadSafeFunction::New(env, info[0].As<Napi::Function>(), "cb", 0, 1);
  }
  void Stop(const Napi::CallbackInfo& info) {
    if (tsfn_) { tsfn_.Release(); } // MUST release or callback lives forever
  }
  ~EventEmitter() {
    if (tsfn_) { tsfn_.Release(); } // also in destructor
  }
};
```

## Common leak patterns

| Pattern | Fix |
|---------|-----|
| Raw pointer with early return / exception | `unique_ptr` |
| Holding Buffer data pointer without reference | `Napi::Persistent(buffer)` |
| ThreadSafeFunction never Released | Call `Release()` when done |
| Large malloc without `AdjustExternalMemory` | Report to V8 |
| Circular JS↔C++ references | Use `Napi::Weak(obj)` for one side |

## Debugging leaks

```bash
# Build with debug symbols
node-gyp rebuild --debug

# Valgrind
valgrind --leak-check=full node --expose-gc test.js

# AddressSanitizer (in binding.gyp)
"cflags": ["-fsanitize=address"],
"ldflags": ["-fsanitize=address"]
# then: ASAN_OPTIONS=detect_leaks=1 node test.js
```

## References

- https://nodejs.org/api/buffer.html
- https://clang.llvm.org/docs/AddressSanitizer.html
