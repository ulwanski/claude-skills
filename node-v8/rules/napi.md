---
name: napi
description: N-API C addon development, ABI stability, async workers
metadata:
  tags: napi, native-addons, abi-stability, async-workers
---

# N-API (Node-API)

N-API provides ABI-stable bindings — addons compile once and work across Node.js versions.

## Minimal addon structure

```c
// binding.gyp: "defines": ["NAPI_VERSION=8"]

#include <node_api.h>

napi_value Add(napi_env env, napi_callback_info info) {
  size_t argc = 2;
  napi_value args[2];
  napi_get_cb_info(env, info, &argc, args, NULL, NULL);

  double a, b;
  napi_get_value_double(env, args[0], &a);
  napi_get_value_double(env, args[1], &b);

  napi_value result;
  napi_create_double(env, a + b, &result);
  return result;
}

napi_value Init(napi_env env, napi_value exports) {
  napi_value fn;
  napi_create_function(env, NULL, 0, Add, NULL, &fn);
  napi_set_named_property(env, exports, "add", fn);
  return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

## Always check status

```c
napi_status status = napi_get_value_double(env, args[0], &value);
if (status != napi_ok) {
  napi_throw_error(env, NULL, "Failed to get number");
  return NULL;
}
```

## Async worker (Promise-based)

```c
typedef struct { napi_async_work work; napi_deferred deferred; char* result; } AsyncData;

void Execute(napi_env env, void* data) {
  AsyncData* d = (AsyncData*)data;
  d->result = doBlockingWork(); // runs on thread pool
}

void Complete(napi_env env, napi_status status, void* data) {
  AsyncData* d = (AsyncData*)data;
  napi_value result;
  napi_create_string_utf8(env, d->result, NAPI_AUTO_LENGTH, &result);
  napi_resolve_deferred(env, d->deferred, result);
  napi_delete_async_work(env, d->work);
  free(d);
}

napi_value RunAsync(napi_env env, napi_callback_info info) {
  AsyncData* d = malloc(sizeof(AsyncData));
  napi_value promise;
  napi_create_promise(env, &d->deferred, &promise);
  napi_value name;
  napi_create_string_utf8(env, "RunAsync", NAPI_AUTO_LENGTH, &name);
  napi_create_async_work(env, NULL, name, Execute, Complete, d, &d->work);
  napi_queue_async_work(env, d->work);
  return promise;
}
```

## ThreadSafeFunction — call JS from any thread

```c
napi_threadsafe_function tsfn;

// From any thread:
napi_call_threadsafe_function(tsfn, data, napi_tsfn_blocking);

// When done:
napi_release_threadsafe_function(tsfn, napi_tsfn_release);
```

## Buffer handling

```c
// Receive Buffer
void* data; size_t length;
napi_get_buffer_info(env, args[0], &data, &length);

// Return Buffer (copies data)
napi_value buf;
napi_create_buffer_copy(env, length, myData, NULL, &buf);

// Return external Buffer (finalizer frees data)
napi_create_external_buffer(env, length, myData,
  [](napi_env, void* data, void*) { free(data); }, NULL, &buf);
```

## References

- https://nodejs.org/api/n-api.html
