---
name: node-addon-api
description: node-addon-api C++ wrapper patterns and best practices
metadata:
  tags: node-addon-api, napi, cpp, native-addons
---

# node-addon-api

C++ wrapper over N-API. Easier type conversions, RAII, exceptions.

## Setup

```bash
npm install node-addon-api
```

```python
# binding.gyp
"include_dirs": ["<!@(node -p \"require('node-addon-api').include\")"],
"defines": ["NAPI_VERSION=8", "NAPI_CPP_EXCEPTIONS"],
"cflags_cc!": ["-fno-exceptions"],
```

## Basic function

```cpp
#include <napi.h>

Napi::Value Add(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();
  if (info.Length() < 2 || !info[0].IsNumber() || !info[1].IsNumber())
    throw Napi::TypeError::New(env, "Expected two numbers");
  return Napi::Number::New(env,
    info[0].As<Napi::Number>().DoubleValue() +
    info[1].As<Napi::Number>().DoubleValue());
}

Napi::Object Init(Napi::Env env, Napi::Object exports) {
  exports.Set("add", Napi::Function::New(env, Add));
  return exports;
}
NODE_API_MODULE(addon, Init)
```

## ObjectWrap — wrapping a C++ class

```cpp
class Counter : public Napi::ObjectWrap<Counter> {
public:
  static Napi::Object Init(Napi::Env env, Napi::Object exports) {
    auto func = DefineClass(env, "Counter", {
      InstanceMethod("increment", &Counter::Increment),
      InstanceAccessor("value", &Counter::GetValue, nullptr),
    });
    exports.Set("Counter", func);
    return exports;
  }
  Counter(const Napi::CallbackInfo& info)
    : Napi::ObjectWrap<Counter>(info), value_(0) {}
private:
  int value_;
  Napi::Value Increment(const Napi::CallbackInfo& info) {
    return Napi::Number::New(info.Env(), ++value_);
  }
  Napi::Value GetValue(const Napi::CallbackInfo& info) {
    return Napi::Number::New(info.Env(), value_);
  }
};
```

## AsyncWorker with Promise

```cpp
class SleepWorker : public Napi::AsyncWorker {
  int ms_;
  Napi::Promise::Deferred deferred_;
public:
  SleepWorker(Napi::Env env, int ms)
    : Napi::AsyncWorker(env), ms_(ms),
      deferred_(Napi::Promise::Deferred::New(env)) {}
  Napi::Promise Promise() { return deferred_.Promise(); }
  void Execute() override {
    std::this_thread::sleep_for(std::chrono::milliseconds(ms_));
  }
  void OnOK() override {
    deferred_.Resolve(Napi::String::New(Env(), "done"));
  }
  void OnError(const Napi::Error& e) override {
    deferred_.Reject(e.Value());
  }
};

Napi::Value Sleep(const Napi::CallbackInfo& info) {
  auto* w = new SleepWorker(info.Env(), info[0].As<Napi::Number>().Int32Value());
  auto promise = w->Promise();
  w->Queue();
  return promise;
}
```

## TypedThreadSafeFunction

```cpp
using TSFN = Napi::TypedThreadSafeFunction<void, int, [](Napi::Env env, Napi::Function cb, void*, int* data) {
  cb.Call({ Napi::Number::New(env, *data) });
  delete data;
}>;

TSFN tsfn = TSFN::New(env, callback, "work", 0, 1);
// From any thread:
tsfn.BlockingCall(new int(42));
// When done:
tsfn.Release();
```

## Report external memory to V8

```cpp
// Constructor
Napi::MemoryManagement::AdjustExternalMemory(env, +size_);
// Destructor callback
Napi::MemoryManagement::AdjustExternalMemory(env, -size_);
```

## References

- https://github.com/nodejs/node-addon-api
