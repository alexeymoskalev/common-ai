# Testing — Gateway & Reactive Code

Complements [java.md](java.md) §4 (JUnit 5 / AssertJ / Mockito, `snake_case` test method names, prefer real objects over mocks). This file covers what is specific to the gateway and to RxJava.

## 1. Reactive unit tests: `TestObserver` / `TestSubscriber`

Never `Thread.sleep`, never `blockingGet()` in an assertion.

```java
policy.onRequest(ctx)
    .test()
    .awaitDone(10, TimeUnit.SECONDS)
    .assertComplete()
    .assertNoErrors();

request.messages()
    .test()
    .awaitDone(10, TimeUnit.SECONDS)
    .assertValueCount(3)
    .assertValueAt(0, m -> m.content().toString().equals("first"));
```

For timers and delays, install a `TestScheduler` instead of waiting:

```java
@BeforeEach void setUp() {
    RxJavaPlugins.setComputationSchedulerHandler(s -> testScheduler);
    RxJavaPlugins.setIoSchedulerHandler(s -> testScheduler);
}
@AfterEach void tearDown() { RxJavaPlugins.reset(); }
```

`RxJavaPlugins.reset()` in `@AfterEach` is mandatory — the handlers are global and leak into other test classes otherwise. (See `DefaultApiReactorTest`, `SyncApiReactorTest`.)

## 2. Gateway integration tests

Module: `gravitee-apim-integration-tests`, SDK: `gravitee-apim-gateway/gravitee-apim-gateway-tests-sdk`.

```java
@GatewayTest
@DeployApi({ "/apis/v4/http/api.json" })
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class MyFeatureV4IntegrationTest extends AbstractGatewayTest {

    @Override
    public void configurePolicies(Map<String, PolicyPlugin> policies) {
        policies.put("add-header", PolicyBuilder.build("add-header", AddHeaderPolicy.class));
    }

    @Override
    public void configureEntrypoints(Map<String, EntrypointConnectorPlugin<?, ?>> entrypoints) {
        entrypoints.putIfAbsent("http-proxy", EntrypointBuilder.build("http-proxy", HttpProxyEntrypointConnectorFactory.class));
    }

    @Override
    public void configureEndpoints(Map<String, EndpointConnectorPlugin<?, ?>> endpoints) {
        endpoints.putIfAbsent("http-proxy", EndpointBuilder.build("http-proxy", HttpProxyEndpointConnectorFactory.class));
    }

    @Test
    void should_get_200(HttpClient httpClient) {
        wiremock.stubFor(get("/endpoint").willReturn(ok("response from backend")));

        httpClient.rxRequest(HttpMethod.GET, "/test")
            .flatMap(HttpClientRequest::rxSend)
            .flatMapPublisher(response -> {
                assertThat(response.statusCode()).isEqualTo(200);
                return response.toFlowable();
            })
            .test()
            .await()
            .assertComplete()
            .assertNoErrors();

        wiremock.verify(1, getRequestedFor(urlPathEqualTo("/endpoint")));
    }
}
```

Rules that are easy to get wrong:

- **`HttpClient` is a test-method parameter**, resolved by the SDK and already pointed at the running gateway. Do not build your own client or hardcode a port. It is `io.vertx.rxjava3.core.http.HttpClient` (the Rx one).
- `wiremock` is an inherited field — the backend the deployed API's endpoint points at. Configure extra behaviour by overriding `configureWireMock(WireMockConfiguration)`.
- Every policy, entrypoint and endpoint the API definition references **must** be registered in the matching `configureXxx` override, otherwise deployment fails. Use `putIfAbsent` for the standard connectors (a parent class may already have added them) and `put` for your own fakes.
- `configurePolicies` registers a bare class; `loadPolicy(manifest, policies)` registers it as a real plugin — needed only when the policy has a `PolicyContext` initializer.
- Definitions are JSON files under `src/test/resources/apis/...`; `@GatewayTest(configFolder = "…")` points at a `src/test/resources`-relative folder containing `config/gravitee.yml`. Tune runtime settings via `configureGateway(GatewayConfigurationBuilder)`.
- `@GatewayTest` is `@Inherited` and `PER_CLASS` — the gateway starts once per class. Use `@Nested` classes with their own `@DeployApi` for scenario groups, as `ApiProductV4IntegrationTest` does.
- Other deployment annotations: `@DeployOrganization`, `@DeploySharedPolicyGroups`, `@DeployApiProducts`, `@InjectApi`.
- Test fakes live in `io.gravitee.apim.integration.tests.fake` and in the SDK's `policy/fakes` + `connector/fakes` packages — reuse them before writing another one.

Running them (this module is **not** in the default reactor):

```bash
mvn -U -T2C -am -pl gravitee-apim-integration-tests clean install -Dskip.validation=true -DskipTests -Dbundle=dev
mvn -fae -pl gravitee-apim-integration-tests install -Dbundle=dev
```

## 3. `gravitee-apim-rest-api` core tests

The hexagonal core is tested with real domain objects and in-memory adapters, not mocks. Those adapters live in the **default-package directory** `gravitee-apim-rest-api-service/src/test/java/inmemory/` (`AbstractCrudServiceInMemory`, `AbstractQueryServiceInMemory`, `…InMemory`). Check for an existing one before mocking a port.

## 4. Before running any test

```bash
mvn prettier:write -pl <module>
```

The `validate` phase runs Prettier, the license check and ArchUnit rules before compiling, so a formatting slip fails the run before a single test executes.

Run tests from the **parent** module so siblings compile:

```bash
mvn -f gravitee-apim-repository/pom.xml test -pl gravitee-apim-repository-elasticsearch -Dtest=SomeTest -am
```
