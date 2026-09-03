# Coding Guidelines for gravitee-apim-rest-api

## API-first: OpenAPI before JAX-RS

Use this when you add or change **HTTP endpoints** (paths, operations, request/response bodies) in any of these Maven modules (paths in the table are relative to `gravitee-apim-rest-api/`):

| Area | Maven module (path suffix) | OpenAPI spec(s) |
| --- | --- | --- |
| **Management API v2** | `gravitee-apim-rest-api-management-v2/gravitee-apim-rest-api-management-v2-rest` | `src/main/resources/openapi/openapi-*.yaml` — several specs by area (e.g. `openapi-apis.yaml`, `openapi-api-products.yaml`, `openapi-environments.yaml`) |
| **Kafka Explorer** | `gravitee-apim-rest-api-kafka-explorer` | `src/main/resources/openapi/openapi-kafka-explorer.yaml` |
| **Portal API** | `gravitee-apim-rest-api-portal/gravitee-apim-rest-api-portal-rest` | `src/main/resources/portal-openapi.yaml` |
| **Automation API** | `gravitee-apim-rest-api-automation/gravitee-apim-rest-api-automation-rest` | `src/main/resources/open-api.yaml`|

**What happens**

1. **Edit OpenAPI** — paths, operations, `components.schemas`, etc. For **v2**, change the YAML that already holds similar routes/schemas (e.g. product APIs → `openapi-api-products.yaml`). **Portal** and **Kafka Explorer** each have a single spec file (see table).
2. **Compile the module** — e.g. `mvn -pl gravitee-apim-rest-api/<module-path> compile`. The `openapi-generator-maven-plugin` runs with `generateModels=true` and `generateApis=false`, so **Java model/DTO classes** are generated into `target/` (not hand-written in `src/main/java`).
3. **Implement JAX-RS** — resource classes and mappers import those generated types (e.g. `io.gravitee.rest.api.management.v2.rest.model.*`, `io.gravitee.rest.api.kafkaexplorer.rest.model.*`, `io.gravitee.rest.api.portal.rest.model.*`; v2 also uses subpackages such as `...model.analytics.engine` / `...model.logs.engine` where the POM overrides `modelPackage`).

**Rule for new endpoints:** always **API-first** — update the right OpenAPI file, run Maven on that module so generation runs, then add or change resources. Starting from the resource skips step 2, so the model classes do not exist yet and tooling breaks.
## Hexagonal core: `io.gravitee.apim.core`

New business logic goes in `gravitee-apim-rest-api-service/src/main/java/io/gravitee/apim/core/<domain>/`, whose sub-packages are **snake_case** (unusual for Java, but consistent across the whole core):

```
core/<domain>/use_case/        *UseCase        — annotated @UseCase
core/<domain>/domain_service/  *DomainService  — annotated @DomainService
core/<domain>/crud_service/    ports: create/read/update/delete
core/<domain>/query_service/   ports: read-only queries
core/<domain>/service_provider/ ports to external systems
core/<domain>/model/           domain models
core/<domain>/exception/       domain exceptions
```

Adapters implementing those ports live under `io.gravitee.apim.infra`. Both annotations (`io.gravitee.apim.core.UseCase` / `.DomainService`) are declared in the **`gravitee-apim-common`** module, not here.

**Component scanning is restricted.** `UsecaseSpringConfiguration` scans `basePackages = { "io.gravitee.apim.core" }` with an annotation include-filter on `@UseCase` (and `CoreServiceSpringConfiguration` likewise for `@DomainService`). A use case placed outside `io.gravitee.apim.core`, or missing its annotation, is simply never registered as a bean — with no error at startup.

The older `io.gravitee.rest.api.service.*` service layer still exists. Extend it only when modifying existing behaviour there; new features go in the core.

Tests for the core use real domain objects plus the in-memory adapters in `src/test/java/inmemory/` — see [`.agent-rules/gravitee-testing.md`](../.agent-rules/gravitee-testing.md) §3.
