# Gravitee APIM 4 — Architecture & Definition Model

Applies when touching `gravitee-apim-definition`, `gravitee-apim-gateway`, `gravitee-apim-plugin`, or any code that reads/writes a v4 API definition.

Repo baseline: APIM **4.12.14**, Java **21** (`.sdkmanrc`, *not* the "expect 17" line in `CONTRIBUTING.adoc`), Vert.x **5.0.12**, `gravitee-gateway-api` **6.3.0**.

## 1. Module map — where code belongs

| Module | Holds |
| --- | --- |
| `gravitee-apim-definition` | The API definition POJOs (`io.gravitee.definition.model.v4.*`) + `GraviteeMapper` (Jackson). No runtime logic. |
| `gravitee-apim-gateway` | The runtime: reactors, policy chains, flow resolution, HTTP/TCP layers (`io.gravitee.gateway.*`). |
| `gravitee-apim-plugin` | In-tree connector/reactor plugins (entrypoint, endpoint, apiservice, reactor). |
| `gravitee-apim-rest-api` | Management/Portal/Automation APIs + the hexagonal core (`io.gravitee.apim.core`, `io.gravitee.apim.infra`). |
| `gravitee-apim-repository` | Repository ports + mongodb/jdbc/redis/elasticsearch adapters, upgraders. |
| `gravitee-apim-integration-tests` | Gateway integration tests. **Not** in the default reactor. |

`gravitee-apim-integration-tests` is only in the `integration-tests-modules` profile (root `pom.xml`), while `all-modules` is `activeByDefault`. A root `mvn test` does **not** run gateway integration tests — never report them as passing after a root build.

## 2. `ApiType` has eight constants — never switch on three

`io.gravitee.definition.model.v4.ApiType`: `A2A_PROXY`, `AUTHZ`, `EDGE`, `LLM_PROXY`, `MCP_PROXY`, `MESSAGE`, `NATIVE`, `PROXY`.

Docs still frame v4 as "proxy vs message vs native"; the code does not. An `if`/`switch` covering only those three silently drops LLM/MCP/A2A/AUTHZ/EDGE APIs, and the gateway sync mappers throw `IllegalArgumentException("Unsupported ApiType …")` at **deploy time**, not compile time.

Adding a new `ApiType` constant means updating all three sync mappers:
`gravitee-apim-gateway-services-sync/.../process/{repository,local,distributed}/mapper/ApiMapper.java`.

## 3. Same simple name, two different enums

`ApiType`, `ConnectorMode` and `ListenerType` each exist **twice**:

- `io.gravitee.definition.model.v4.*` — the definition POJO side. Used by mappers, CRUD, validation.
- `io.gravitee.gateway.reactive.api.*` — the runtime/connector SPI side (from the `gravitee-gateway-api` jar). Used by `Connector`, `ConnectorFactory`, reactors.

They are not interchangeable and their constant order/labels differ. Always check what the surrounding class's other imports use before adding one.

## 4. Definition classes: there is no `MessageApi`

`AbstractApi` is polymorphic on the `type` discriminator with only three concrete subtypes:

```java
@JsonSubTypes({
  @JsonSubTypes.Type(value = Api.class,       name = PROXY_LABEL),   // "proxy"
  @JsonSubTypes.Type(value = Api.class,       name = MESSAGE_LABEL), // "message"
  @JsonSubTypes.Type(value = NativeApi.class, name = NATIVE_LABEL),  // "native"
  @JsonSubTypes.Type(value = EdgeApi.class,   name = EDGE_LABEL),    // "edge"
})
```

A message API is `io.gravitee.definition.model.v4.Api` with `type=message` and message-capable connectors. `MessageApi`, `ProxyApi`, `HttpApi`, `PlanSecurityType` and `ShardingTag` **do not exist** — do not invent them.

## 5. Flow field names ≠ execution phase names

| `v4.flow.Flow` fields | `v4.nativeapi.NativeFlow` fields | `io.gravitee.gateway.reactive.api.ExecutionPhase` |
| --- | --- | --- |
| `request`, `response`, `publish`, `subscribe` | `entrypointConnect`, `interact`, `subscribe`, `publish` | `REQUEST`, `MESSAGE_REQUEST`, `RESPONSE`, `MESSAGE_RESPONSE`, `ENTRYPOINT_CONNECT` |

The bridge is **`PUBLISH → MESSAGE_REQUEST`** and **`SUBSCRIBE → MESSAGE_RESPONSE`** (see `DefaultSharedPolicyGroupReactor`). Inverting it inverts the whole message pipeline. `flow.getMessageRequest()` and `ExecutionPhase.PUBLISH` do not exist. `NativeFlow` has no `request`/`response`, so native flows cannot reuse HTTP flow code.

For shared policy groups, `SharedPolicyGroup.Phase.MESSAGE_REQUEST` / `MESSAGE_RESPONSE` are `@Deprecated` — new code emits `PUBLISH` / `SUBSCRIBE` (a MongoDB upgrader exists to rewrite the old values).

## 6. Three parallel execution trees in `gravitee-apim-gateway`

Same simple names (`FlowChain`, `FlowChainFactory`, `ApiFlowResolver`, `HttpPolicyChainFactory`) exist in all three. Editing the wrong one produces a change that never runs.

| Package | Engine |
| --- | --- |
| `io.gravitee.gateway.handlers.api.*`, `io.gravitee.gateway.flow.*` | Legacy v3 engine |
| `io.gravitee.gateway.reactive.handlers.api.*` (no `v4` segment), `io.gravitee.gateway.reactive.policy.*` | Reactive engine running **v2** definitions in emulation mode — reads `flow.getPre()` / `getPost()` |
| `io.gravitee.gateway.reactive.handlers.api.v4.*`, `io.gravitee.gateway.reactive.v4.policy.*` | The real **v4** path — reads `flow.getRequest()` / `getResponse()` |

Before editing, confirm which one your API definition actually goes through.

## 7. `configuration` fields are raw JSON strings

Every `configuration` / `sharedConfiguration` / `sharedConfigurationOverride` on entrypoints, endpoints, endpoint groups, steps and plan security is a `String` annotated `@JsonRawValue`, with a `@JsonSetter(JsonNode)` overload:

```java
@JsonRawValue
private String configuration;
@JsonSetter public void setConfiguration(final JsonNode c) { if (c != null) this.configuration = c.toString(); }
public void setConfiguration(final String c) { this.configuration = c; }
```

Connector/policy config schemas live in the plugins, so the definition model keeps them opaque. Never try to bind a typed config POJO inside `gravitee-apim-definition`; deserialize it in the plugin with `PluginConfigurationHelper`.

## 8. `GraviteeMapper` never rejects a bad definition

`io.gravitee.definition.jackson.datatype.GraviteeMapper` disables `FAIL_ON_UNKNOWN_PROPERTIES` and `FAIL_ON_INVALID_SUBTYPE`, and enables `READ_UNKNOWN_ENUM_VALUES_USING_DEFAULT_VALUE` + `ACCEPT_CASE_INSENSITIVE_ENUMS`.

An unknown field vanishes and a bad enum label falls back to the `@JsonEnumDefaultValue`. A test that only asserts "`readValue` did not throw" proves nothing — **assert on the resulting field values**.

Also: `@JsonProperty("gravitee")` exists only on the **v2** `io.gravitee.definition.model.Api`. In a v4 payload the key is `definitionVersion`; a stray `"gravitee": "4.0.0"` is silently ignored.

## 9. `PlanSecurity.type` is a `String` — do not normalise it

The gateway resolves the security policy plugin id by `type.toLowerCase().replaceAll("_", "-")` (`SecurityPolicyFactory`), and `AbstractPlan` compares case-insensitively against **both** `"API_KEY"` and `"api-key"`. "Cleaning up" the value changes which policy loads — and an unresolvable one only logs a WARN, so the plan is silently skipped.

## 10. Sharding tags

Tags are `Set<String> tags` on `AbstractApi`, `AbstractPlan` and `AbstractFlow` — independently. Matching goes through `GatewayConfiguration.hasMatchingTags(Set<String>)`; plan filtering through `AbstractApiDeployer.filterShardingTag(...)`, where **null/empty means deploy everywhere**. Do not re-implement this.

## 11. What is NOT in this repo

Only these ship here:

- entrypoints: `http-proxy`, `tcp-proxy`
- endpoints: `http-proxy`, `tcp-proxy`, `mock`
- reactors: `DefaultApiReactorFactory` (V4 + PROXY + HTTP listener), `TcpApiReactorFactory` (V4 + PROXY + TCP listener)

Kafka / MQTT5 / RabbitMQ / SSE / Webhook / WebSocket / HTTP-GET-POST connectors and the message & native-Kafka reactors live in **separate plugin repositories**, loaded at runtime through the `ConnectorFactory` / `ReactorFactory` SPIs. If a task points at one of them, say it is out of tree — do not write a replacement here.

Likewise `ListenerType.KAFKA` only exists on the native branch: `Listener` (used by `v4.Api`) declares subtypes for HTTP, SUBSCRIPTION and TCP only; `KafkaListener` hangs off `NativeListener`. `listener.getType() == ListenerType.KAFKA` compiles inside HTTP code paths but can never be true there.

## 12. Build gates that fail before compilation

Bound to the `validate` phase in the root `pom.xml` (all skippable with `-Dskip.validation=true`, which CI does not do):

- `license-maven-plugin check` — Apache-2.0 header on every `src/main/java` and `src/test/java` file (`src/main/resources`, `src/test/resources`, `.` -prefixed dirs and the frontends are excluded). Copy the header block from a neighbouring file.
- `prettier-maven-plugin check` — Google Java Style over `src/{main,test}/**/*.java` and `docker/**/docker-compose.yml`. Fix with `mvn prettier:write -pl <module>`.
- `gravitee-archrules-maven-plugin` — executions `global-logging-check` and `execution-context-logging-check`. This is why `@CustomLog` + `ctx.withLogger(log)` is a hard requirement, not a style preference (see [gravitee-v4-reactive-api.md](gravitee-v4-reactive-api.md)).

Useful commands:

```bash
mvn clean install -T 2C -DskipTests=true -Dskip.validation=true      # fast full build
mvn prettier:write -pl <module>                                       # before any test run
mvn -f <top-module>/pom.xml test -pl <child-module> -Dtest=... -am     # single test, siblings compiled
mvn -fae -pl gravitee-apim-integration-tests install -Dbundle=dev      # gateway integration tests
```
