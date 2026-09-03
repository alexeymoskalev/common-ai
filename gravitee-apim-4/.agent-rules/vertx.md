# Vert.x 5 & RxJava 3

Applies to every backend module that touches the gateway runtime, a verticle, an HTTP client/server, or a reactive chain. Vert.x version is pinned at the root `pom.xml` (`<vertx.version>5.0.12`).

## 1. The schedulers are not the schedulers you think

`HttpProtocolVerticle.start()` rewires RxJava globally onto Vert.x:

```java
RxJavaPlugins.setComputationSchedulerHandler(s -> RxHelper.scheduler(vertx));         // event loop
RxJavaPlugins.setIoSchedulerHandler(s -> RxHelper.blockingScheduler(vertx));          // vertx.executeBlocking
RxJavaPlugins.setNewThreadSchedulerHandler(s -> RxHelper.scheduler(vertx));           // event loop
RxJavaPlugins.setErrorHandler(this::logGlobalErrors);
```

Consequences you must code around:

- **`Schedulers.io()` means `vertx.executeBlocking`**, and `Schedulers.computation()` means *the Vert.x event loop*. The standard RxJava mental model does not apply.
- **Never capture a `Scheduler` in a `static final` field or in a constructor that runs at class-load time.** The handlers are installed when the verticle starts; a scheduler resolved earlier is the plain RxJava one and loses the Vert.x context. Resolve lazily, at API-deploy time or at subscription time — see the `PEN-88` comments in `HttpPolicyFactory` and `PolicyAdapter`.
- The idiom for offloading blocking work and coming back to the right context is:

  ```java
  exec.subscribeOn(Schedulers.io())          // → executeBlocking (worker)
      .observeOn(Schedulers.computation());  // → back to the event loop
  ```

  Coming back matters: downstream Vert.x calls (backend HTTP request, response write) must run on the event loop.
- `RxJavaPlugins.setErrorHandler` is already set globally. Do not install your own; do not swallow errors to "avoid" it.

Two different `RxHelper` classes are in play — do not mix them up:
`io.vertx.rxjava3.core.RxHelper` (schedulers) vs `io.gravitee.common.utils.RxHelper` (`retryExponentialBackoff(initialDelay, maxDelay, unit, factor)`, used by the sync managers). For retry-with-backoff, use the Gravitee one instead of hand-rolling `retryWhen`.

## 2. Never block the event loop

Forbidden on any request path (policies, connectors, invokers, processors, hooks):

- `.blockingGet()`, `.blockingAwait()`, `.blockingFirst()`, `.blockingSubscribe()`
- JDBC / file I/O / `Thread.sleep` / synchronous HTTP clients
- long CPU loops

`blockingAwait()` does appear in a few **lifecycle** methods (`DefaultApiReactor.doStart/doStop`, `ApiProductManagerImpl.register`) — those run at deploy time, not per request. That is the only acceptable use; do not copy it into request handling.

Wrap unavoidable blocking work in `Completable.fromRunnable(...)` / `Single.fromCallable(...)` and `subscribeOn(Schedulers.io())`.

## 3. Cold sources and lazy side effects

- Wrap side-effecting chain construction in `Completable.defer(...)` / `Flowable.defer(...)` so the work happens per subscription, not when the chain is built. The mock endpoint connector's `publish` does exactly this.
- Use `Completable.fromRunnable` / `Single.fromCallable` for synchronous work — never do it in the enclosing method body and then return `Completable.complete()`.
- **A returned Rx type that is not returned up the chain is never subscribed.** If a method's contract is `Completable`, every branch must return the composed chain; `andThen` / `concatWith` to sequence, not statements.
- Prefer `concatMap` over `flatMap` when order matters — messages and body chunks are ordered streams and `flatMap` will interleave them.

## 4. Bodies and messages are single-subscription

A request/response body (`Maybe<Buffer>`) and a message stream (`Flowable<Message>`) are backed by a live network stream. Subscribing twice either replays nothing or throws. Use the `onBody` / `onChunks` / `onMessages` / `onMessage` transformer methods, which install exactly one subscription for you, instead of reading and re-setting.

Backpressure on `chunks()` / `messages()` is real — do not `.toList()` a message stream or buffer an unbounded body.

## 5. Vert.x core vs rxjava3 API

Both are used in this repo:

| | Package | Where |
| --- | --- | --- |
| Core (Future-based) | `io.vertx.core.*` | options, `HttpMethod`, `MultiMap`, `SocketAddress`, `JsonObject`, low-level server plumbing |
| Rx bridge | `io.vertx.rxjava3.core.*` | anything composed into an Rx chain: `HttpClient`, `HttpServerRequest`, `NetSocket`, `Vertx` |

Conversion: `rxObject.getDelegate()` → core object; `io.vertx.rxjava3.core.Vertx.newInstance(coreVertx)` → rx wrapper. Do not create a second `Vertx` instance — obtain the existing one by Spring injection or from `AbstractVerticle.vertx`.

**Buffers**: `io.gravitee.gateway.api.buffer.Buffer` is the gateway's own type and is what the reactive API uses. `io.vertx.core.buffer.Buffer` only appears in the low-level HTTP layer. Do not mix them across the policy/connector boundary.

## 6. Vert.x 5 specifics

- The callback API is gone: there is no `Handler<AsyncResult<T>>` overload any more. Everything returns `io.vertx.core.Future<T>` (core) or an Rx type (rxjava3 bridge). Code or docs showing `client.request(opts, ar -> {...})` is Vert.x 4 and will not compile.
- Verticles: extend `io.vertx.rxjava3.core.AbstractVerticle` and override `rxStart()` / `rxStop()` when the startup is asynchronous; plain `start()` / `stop()` only for synchronous setup. The in-tree verticles (`HttpProtocolVerticle`, `TcpProtocolVerticle`, `EndpointHealthcheckVerticle`, `EndpointDiscoveryVerticle`) are the reference implementations.
- Exceptions escaping a Vert.x context are caught by `Vertx.currentContext().exceptionHandler(...)`, already installed by `HttpProtocolVerticle`.
- Timers: `vertx.setTimer` / `setPeriodic` take a `Handler<Long>` and must be cancelled in `stop()`. Prefer `Flowable.timer(..., Schedulers.computation())` inside a chain — it is cancelled with the subscription.

## 7. Testing reactive code

See [gravitee-testing.md](gravitee-testing.md). Key point specific to schedulers: use `RxJavaPlugins.setComputationSchedulerHandler` / `setIoSchedulerHandler` with a `TestScheduler` in `@BeforeEach` and `RxJavaPlugins.reset()` in `@AfterEach` (as `DefaultApiReactorTest` and `SyncApiReactorTest` do) — never `Thread.sleep` to wait for a timer.
