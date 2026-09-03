# Writing v4 Plugins — Policies, Connectors, API Services

Read together with [gravitee-v4-reactive-api.md](gravitee-v4-reactive-api.md) (the context/body/message API) and [gravitee-v4-architecture.md](gravitee-v4-architecture.md) (what lives where).

## 1. Plugin module anatomy

Reference implementation: `gravitee-apim-plugin/gravitee-apim-plugin-endpoint/gravitee-apim-plugin-endpoint-mock`.

```
src/main/assembly/plugin-assembly.xml         # zip: jar + schemas/ + images/ + lib/ (deps)
src/main/java/…/XxxConnector.java             # the connector/policy itself
src/main/java/…/XxxConnectorFactory.java      # declared as `class=` in plugin.properties
src/main/java/…/configuration/XxxConfiguration.java
src/main/resources/plugin.properties
src/main/resources/schemas/schema-form.json   # UI config form
src/main/resources/images/xxx.svg | .png
src/test/java/…/XxxConnectorTest.java
src/test/java/…/XxxConnectorFactoryTest.java
```

`plugin.properties` — every key matters:

```properties
id=mock
name=Mock
version=${project.version}
description=${project.description}
class=io.gravitee.plugin.endpoint.mock.MockEndpointConnectorFactory
type=endpoint-connector
features=limit,resume
icon=images/mock.svg
moreInfo.description=…
moreInfo.documentationUrl=…
moreInfo.schemaImg=images/mock.png
```

`class` points at the **factory**, not the connector. `id` must equal what the connector's `id()` returns and what the API definition references.

## 2. Policies

A v4 policy implements `io.gravitee.gateway.reactive.api.policy.http.HttpPolicy` — **not** the deprecated `…api.policy.Policy`.

```java
public class ConnectorToHeaderPolicy implements HttpPolicy {

    @Override
    public String id() {
        return "connector-to-header";
    }

    @Override
    public Completable onResponse(HttpPlainExecutionContext ctx) {
        return Completable.fromRunnable(() -> ctx.response().headers().add("X-Foo", "bar"));
    }
}
```

- All four methods are `default` no-ops — override only the phases you handle. Which phases you implement *is* the phase declaration; there is no annotation.
- `onRequest` / `onResponse` take `HttpPlainExecutionContext`; `onMessageRequest` / `onMessageResponse` take `HttpMessageExecutionContext`. One class can implement both pairs.
- A security policy implements `HttpSecurityPolicy` and must provide `Maybe<SecurityToken> extractSecurityToken(HttpPlainExecutionContext)`. The plugin id is derived from `PlanSecurity.type` by `toLowerCase().replaceAll("_","-")`, so the id must match that form.
- Short-circuit with `ctx.interruptWith(new ExecutionFailure(403).key("…").message("…").cause(e))`; keep going with `ctx.warnWith(new ExecutionWarn("…"))`.
- Configuration is deserialized from the raw JSON string with `PluginConfigurationHelper.readConfiguration(MyConfiguration.class, configuration)`; the matching JSON schema goes in `src/main/resources/schemas/schema-form.json`.
- Legacy v3 policies (`io.gravitee.gateway.policy.Policy`, `PolicyChain`, `ReadWriteStream`) still run via `PolicyAdapter`. Do not write new ones; do not "convert" existing ones unless the task asks.

## 3. Entrypoint connectors

Extend `HttpEntrypointSyncConnector` (request/response) or `HttpEntrypointAsyncConnector` (messages), plus a factory implementing `HttpEntrypointSyncConnectorFactory` / `HttpEntrypointAsyncConnectorFactory`.

```java
public class HttpProxyEntrypointConnector extends HttpEntrypointSyncConnector {
    static final Set<ConnectorMode> SUPPORTED_MODES = Set.of(ConnectorMode.REQUEST_RESPONSE);
    static final ListenerType SUPPORTED_LISTENER_TYPE = ListenerType.HTTP;   // gateway.reactive.api.ListenerType

    @Override public String id() { return "http-proxy"; }
    @Override public int matchCriteriaCount() { return Integer.MIN_VALUE; }  // lower = weaker match
    // + matches(ctx), handleRequest(ctx), handleResponse(ctx)  → Completable
}
```

- `matchCriteriaCount()` orders candidates; the fallback/catch-all connector returns `Integer.MIN_VALUE`.
- Async entrypoints must also implement `QosRequirement qosRequirement()`, and get `stopHook` / `emitStopMessage()` / `applyStopHook()` from the base class for graceful drain. Wire `applyStopHook()` into the message stream or the entrypoint will not shut down cleanly.

## 4. Endpoint connectors

Extend `HttpEndpointSyncConnector` (`Completable connect(ctx)`) or `HttpEndpointAsyncConnector`:

```java
public class MockEndpointConnector extends HttpEndpointAsyncConnector {
    @Override public String id() { return "mock"; }
    @Override public Set<ConnectorMode>   supportedModes()          { return Set.of(PUBLISH, SUBSCRIBE); }
    @Override public Set<Qos>             supportedQos()            { return Set.of(NONE, AUTO, AT_LEAST_ONCE, AT_MOST_ONCE); }
    @Override public Set<QosCapability>   supportedQosCapabilities(){ return Set.of(AUTO_ACK, MANUAL_ACK, RECOVER); }
    @Override public Completable subscribe(HttpExecutionContext ctx) { … ctx.response().messages(flow); }
    @Override public Completable publish  (HttpExecutionContext ctx) { … ctx.request().onMessage(…); }
}
```

- `subscribe` = gateway → client (set `ctx.response().messages(...)`). `publish` = client → backend (consume `ctx.request()`).
- The `connect(...)` overloads on the base class still take the deprecated `io.gravitee.gateway.reactive.api.context.http.HttpExecutionContext`; that is the SPI signature — match it, do not change it.
- Declare only QoS levels you genuinely implement. `AT_LEAST_ONCE` / `RECOVER` require honouring the `ATTR_INTERNAL_MESSAGES_RECOVERY_LAST_ID` internal attribute and calling `message.ack()`.
- Base-class failure keys (`FAILURE_ENDPOINT_CONNECTION_FAILED`, `FAILURE_ENDPOINT_PUBLISH_FAILED`, …) exist — use them instead of inventing new strings.

## 5. Factories

```java
@CustomLog
@AllArgsConstructor
public class MockEndpointConnectorFactory implements HttpEndpointAsyncConnectorFactory<MockEndpointConnector> {

    private PluginConfigurationHelper pluginConfigurationHelper;

    @Override public Set<ConnectorMode> supportedModes() { return MockEndpointConnector.SUPPORTED_MODES; }
    @Override public Set<Qos>           supportedQos()   { return MockEndpointConnector.SUPPORTED_QOS; }

    @Override
    public MockEndpointConnector createConnector(DeploymentContext deploymentContext, String configuration, String sharedConfiguration) {
        try {
            return new MockEndpointConnector(pluginConfigurationHelper.readConfiguration(MockEndpointConnectorConfiguration.class, configuration));
        } catch (PluginConfigurationException e) {
            log.error("Can't create connector cause no valid configuration", e);
            return null;
        }
    }
}
```

The factory takes a single `PluginConfigurationHelper` constructor argument (injected by the plugin framework) — keep that shape. **Returning `null` on a bad configuration is the established contract**; do not throw out of `createConnector`.

Entrypoint factories take `(DeploymentContext, String configuration)` — no `sharedConfiguration`. Endpoint factories take all three.

## 6. API services

`io.gravitee.gateway.reactive.api.apiservice.ApiService`: `String id()`, `String kind()`, `Completable start()`, `Completable stop()`, created by an `ApiServiceFactory`. Services are started/stopped by `DefaultApiReactor` at deploy time (that is where a `blockingAwait()` is legitimate — see [vertx.md](vertx.md) §2).

## 7. Before you write a new plugin module

Check it is not one of the out-of-tree connectors listed in [gravitee-v4-architecture.md](gravitee-v4-architecture.md) §11. Kafka, MQTT, SSE, Webhook, WebSocket and HTTP-GET/POST connectors live in separate repositories — adding a stub here is wrong.
