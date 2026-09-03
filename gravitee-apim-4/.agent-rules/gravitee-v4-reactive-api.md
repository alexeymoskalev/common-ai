# Gravitee v4 Reactive Execution API

The API surface every policy, connector, invoker, processor and hook in the v4 gateway talks to. It comes from the external jar `io.gravitee.gateway:gravitee-gateway-api:6.3.0` — the sources are **not** in this tree, so verify signatures with `javap` against the jar in `~/.m2` rather than guessing:

```bash
javap -cp ~/.m2/repository/io/gravitee/gateway/gravitee-gateway-api/6.3.0/gravitee-gateway-api-6.3.0.jar \
  io.gravitee.gateway.reactive.api.context.http.HttpPlainExecutionContext
```

## 1. Use the `context.http` package, not `context`

`io.gravitee.gateway.reactive.api.context.{ExecutionContext, HttpExecutionContext, GenericExecutionContext, MessageExecutionContext, Request, Response, …}` are **all `@Deprecated`**, as is `io.gravitee.gateway.reactive.api.policy.Policy`.

New code uses:

| Deprecated | Use instead |
| --- | --- |
| `…api.context.HttpExecutionContext` | `…api.context.http.HttpPlainExecutionContext` |
| `…api.context.MessageExecutionContext` | `…api.context.http.HttpMessageExecutionContext` |
| `…api.context.GenericExecutionContext` | `…api.context.base.BaseExecutionContext` / `…context.http.HttpBaseExecutionContext` |
| `…api.policy.Policy` | `…api.policy.http.HttpPolicy` |

The deprecated interfaces still appear in older in-tree code (and in `EndpointConnector.connect`, whose SPI signature is fixed) — do not "modernise" those signatures, but never introduce them in new classes.

The context hierarchy:

```
BaseExecutionContext
 └── HttpBaseExecutionContext          metrics(), request(), response()
      ├── HttpPlainExecutionContext    plain HTTP  → policy REQUEST / RESPONSE phases
      └── HttpMessageExecutionContext  messages    → policy MESSAGE_REQUEST / MESSAGE_RESPONSE phases
```

## 2. Attributes

From `BaseExecutionContext`:

```java
<T> T getAttribute(String);      void setAttribute(String, Object);   void putAttribute(String, Object);
<T> List<T> getAttributeAsList(String);   Set<String> getAttributeNames();   <T> Map<String,T> getAttributes();
<T> T getInternalAttribute(String);      void setInternalAttribute(String, Object);
```

Two separate namespaces. **Public** attributes are visible to EL expressions and to users; **internal** attributes are gateway plumbing and are not exposed. Constant names live in `io.gravitee.gateway.reactive.api.context.ContextAttributes` and `InternalContextAttributes` — always use the constants, never string literals:

```java
ctx.getInternalAttribute(InternalContextAttributes.ATTR_INTERNAL_ENDPOINT_CONNECTOR_ID);
```

Other `BaseExecutionContext` members worth knowing: `<T> T getComponent(Class<T>)` (Spring/plugin components), `getTemplateEngine()` (EL), `getTracer()`, `timestamp()`, `remoteAddress()`, `localAddress()`, `tlsSession()`, `warnWith(ExecutionWarn)`.

## 3. Bodies: `Buffer` is Gravitee's, not Vert.x's

The body type is **`io.gravitee.gateway.api.buffer.Buffer`**, never `io.vertx.core.buffer.Buffer`. Create with `Buffer.buffer(...)`.

`HttpPlainRequest` / `HttpPlainResponse`:

```java
Maybe<Buffer>  body();                    // empty Maybe when there is no body
Single<Buffer> bodyOrEmpty();             // never empty — prefer when you always need a buffer
void           body(Buffer);              // replace
Completable    onBody(MaybeTransformer<Buffer, Buffer>);          // transform, buffered
Flowable<Buffer> chunks();                                        // streaming
void           chunks(Flowable<Buffer>);
Completable    onChunks(FlowableTransformer<Buffer, Buffer>);     // transform, streaming
void           contentLength(long);
```

Rules:

- Prefer `onBody` / `onChunks` over `body()` + `body(...)`: they wire the replacement in for you and keep a single subscription.
- Prefer `onChunks` when you do not need the whole payload — `onBody` buffers the entire body in memory.
- **Return the `Completable`.** `onBody(...)` without returning it (or without `andThen`) is never subscribed, so nothing happens. This is the single most common bug in new policy code.
- After changing a body length, set `contentLength(...)` (or remove the header) — otherwise the client hangs or truncates.

`HttpPlainRequest` also has `isWebSocket()`, `webSocket()`, `method(HttpMethod)`.
`HttpPlainResponse` also has `Completable end(HttpBaseExecutionContext)`.

## 4. Messages

`HttpMessageRequest` / `HttpMessageResponse`:

```java
Flowable<Message> messages();
void              messages(Flowable<Message>);
Completable       onMessages(FlowableTransformer<Message, Message>);
Completable       onMessage(Function<Message, Maybe<Message>>);   // per-message; empty Maybe drops it
```

`io.gravitee.gateway.reactive.api.message.Message` (impl: `DefaultMessage`): `id()`, `correlationId()`, `parentCorrelationId()`, `timestamp()`, `error()`, `metadata()`, `headers()`, `content()` / `content(Buffer|String)`, `attribute(String)`, `ack()`.

Returning `Maybe.empty()` from `onMessage` **filters the message out** — that is the idiomatic way to drop one, not throwing.

## 5. Interrupting the flow

```java
// HttpPlainExecutionContext
Completable    interrupt();
Completable    interruptWith(ExecutionFailure);
Maybe<Buffer>  interruptBody();
Maybe<Buffer>  interruptBodyWith(ExecutionFailure);
// HttpMessageExecutionContext
Flowable<Message> interruptMessagesWith(ExecutionFailure);
Maybe<Message>    interruptMessageWith(ExecutionFailure);
// BaseExecutionContext
void warnWith(ExecutionWarn);
```

Pick the variant matching the type your operator must return — `interruptWith` inside a `flatMap` on a `Maybe<Buffer>` will not compile; use `interruptBodyWith`.

`ExecutionFailure` is a fluent builder: `statusCode(int)`, `key(String)`, `message(String)`, `parameters(Map)`, `contentType(String)`, `cause(Throwable)`.
`ExecutionWarn` takes the key in its constructor: `new ExecutionWarn("KEY").message(...).cause(e)`.

Conventions (see also [java.md](java.md) §2):

- `interruptWith` = the request stops. `warnWith` = it continues.
- Always `.cause(e)` when an exception is in hand.
- Keys are specific and actionable: `"CORS_PREFLIGHT_FAILED"`, `"RATE_LIMIT_TOO_MANY_REQUESTS"` — not `"ERROR"`.

## 6. Logging is enforced by ArchUnit

`gravitee-archrules-maven-plugin` runs at `validate` with two executions (`global-logging-check`, `execution-context-logging-check`). A build failure here is not a style nit.

```java
@CustomLog                       // Lombok — not LoggerFactory.getLogger(...)
public class MyPolicy implements HttpPolicy {
    Completable onRequest(HttpPlainExecutionContext ctx) {
        ctx.withLogger(log).debug("…");   // whenever a context is in scope
    }
}
```

`withLogger` is a `default` method on `BaseExecutionContext` returning an `org.slf4j.Logger` that carries the request context. The gateway test SDK (`io.gravitee.apim.gateway.tests.sdk..`) is excluded from the check.

## 7. Everything returns an Rx type — never block

| Extension point | Signature |
| --- | --- |
| `HttpPolicy` | `Completable onRequest/onResponse(HttpPlainExecutionContext)`, `Completable onMessageRequest/onMessageResponse(HttpMessageExecutionContext)` |
| `HttpSecurityPolicy` | `Maybe<SecurityToken> extractSecurityToken(HttpPlainExecutionContext)`, `Single<Boolean> wwwAuthenticate(...)` |
| `EntrypointConnector` | `boolean matches(...)`, `Completable handleRequest/handleResponse(...)` |
| `EndpointConnector` | `Completable connect(...)` |
| `HttpEndpointAsyncConnector` | `Completable subscribe/publish(HttpExecutionContext)` |
| `ApiService` | `String id()`, `String kind()`, `Completable start()`, `Completable stop()` |

See [vertx.md](vertx.md) for what "never block" means concretely on the event loop.
