# Лицензионные гейты Gravitee APIM 4.12.14 — точки отключения

Полный индекс точек, в которых backend-код APIM проверяет лицензию.
Для каждой точки указан класс:строка, что делает проверка и как её отключить (реализация).

> Оглавление
> 1. [Жёсткая валидация плагинов API](#01)
> 2. [Каталог плагинов (флаг deployed)](#02)
> 3. [Гейты отдельных фич](#03)
> 4. [Флаг plugin.deployed()](#04)
> 5. [Источник лицензии](#05)

---

<a id="01"></a>
## 01 · Жёсткая валидация плагинов API

Единый механизм: `LicenseManager.validatePluginFeatures(orgId, plugins)` валидирует список
плагинов из определения API против лицензии организации. Без отказа от проверки API просто
не разворачивается.

### `ApiLicenseServiceImpl.java:58`
`validatePluginFeatures(orgId, plugins)` — V2 и все типы V4. FEDERATED и FEDERATED_AGENT
проходят валидацию всегда.
**Реализация:** сделать метод no-op — убрать вызов `licenseManager.validatePluginFeatures(...)`
или сразу `return` в начале метода.

### `ApiLicenseServiceImpl.java:59` (–62)
Перевод исключений: `ForbiddenFeatureException` (со списком запрещённых плагинов) и
`InvalidLicenseException`.
**Реализация:** не перевыбрасывать — обернуть в try/catch и логировать, либо не вызывать
валидацию сверху (строка 58).

### `ApiManagerImpl.java:201` (–207)
`doRegister()` — валидация перед развертыванием API в реакторе (gateway).
**Реализация:** закомментировать блок проверки лицензии перед `register()`.

### `ApiManagerImpl.java:208` (–211)
Обработка отказа: `log.warn` и `return false`. API молча остаётся неразвёрнутым.
**Реализация:** вместо `return false` продолжать регистрацию (удалить early-return, оставить
только warn или убрать обработку).

### `SharedPolicyGroupManagerImpl.java:141` (–147)
Та же валидация для плагинов Shared Policy Group при регистрации на шлюзе.
**Реализация:** убрать вызов `licenseManager.validatePluginFeatures(...)` в `doRegister()`.

### `SharedPolicyGroupManagerImpl.java:148` (–155)
Отказ по той же схеме: предупреждение в лог и `false`.
**Реализация:** убрать early-return, продолжить регистрацию.

### `ApiProductManagerImpl.java:88` (–95)
Исключение из правила: проверяется не-плагин, а одна фича перед регистрацией API Product.
Возвращает `Completable.complete()`, то есть отказ выглядит как успех.
**Реализация:** фича `apim-api-products` — отключить в `LicenseDomainService:54` (см. раздел 03).

### `ApiResource.java` (management v2) — `:673`, `:684`, `:708`, `:836`
Вызовы `checkLicense` на эндпоинтах:
- `:673` — `POST /apis/{apiId}/deployments`
- `:684` — `GET /deployments/current`
- `:708` — `GET /deployments/_verify` (возвращает `ok: false` при ошибке)
- `:836` — `POST /apis/{apiId}/_start`
**Реализация:** убрать вызовы `checkLicense(...)` в этих методах ресурса.

---

<a id="02"></a>
## 02 · Каталог плагинов (флаг deployed)

Плагин остаётся видимым, но получает `deployed = false`. Формула:
`plugin.isDeployed() && license.isFeatureEnabled(plugin.getFeature())`.

### `PluginFilterByLicenseDomainService.java:34` (и `:37`)
Центральная точка: `getOrganizationLicenseOrPlatform(orgId)`, затем пересчёт `deployed`
для каждого плагина набора.
**Реализация:** всегда возвращать `deployed = true` (игнорировать результат `isFeatureEnabled`).

### `GetPolicyPluginsUseCase.java:40`
Список policy-плагинов для консоли.
**Реализация:** не пропускать через `PluginFilterByLicenseDomainService` (или форсировать
`deployed=true`).

### `GetResourcePluginsUseCase.java:43`
Список resource-плагинов.
**Реализация:** аналогично — отключить фильтрацию по лицензии.

### `GetResourcePluginUseCase.java:39`
Один resource-плагин по идентификатору.
**Реализация:** аналогично.

### `GetEntrypointPluginsUseCase.java:40` (–43)
Entrypoint-коннекторы.
**Реализация:** аналогично.

### `GetEndpointPluginsUseCase.java:40`
Endpoint-коннекторы.
**Реализация:** аналогично.

### `PoliciesResource.java:78` (и `:136`)
`GET /policies` — держит копию логики domain service: лицензия берётся на строке 78,
пересчёт `deployed` — на 136.
**Реализация:** в обоих точках форсировать `deployed=true` (или убрать обращение к
лицензии).

### `ResourcesResource.java:79`, `:91`, `:113`
`GET /resources` — дублирует логику; лицензия прокидывается как `Function<String, Boolean>`.
**Реализация:** заменить функцию на всегда-истина (`feature -> true`).

---

<a id="03"></a>
## 03 · Гейты отдельных фич

Аннотация `@GraviteeLicenseFeature` на методе ресурса + прямые вызовы `isFeatureEnabled`.
Отказ — `ForbiddenFeatureException` (HTTP 403/404 в зависимости от обработчика).

### `GraviteeLicenseFilter.java:46`, `:51`, `:53`
JAX-RS фильтр, исполняющий аннотацию. Берёт `getPlatformLicense()`, а не организационную
(это отличает его от слоя 02).
**Реализация:** сделать `filter()` no-op (не прерывать цепочку) — тогда все
`@GraviteeLicenseFeature`-аннотированные методы пройдут.

### `TagsResource.java:107`, `:137`, `:163`
Создание/обновление/удаление sharding-тегов. Фича `apim-sharding-tags`.
**Реализация:** убрать проверку `isFeatureEnabled("apim-sharding-tags")`.

### `ApiResource.java` (management v1): `:411`
Запуск отладки API. Фича `apim-debug-mode`.
**Реализация:** убрать вызов проверки фичи.

### `ApiAuditResource.java:65`
Журнал аудита на уровне API. Фича `apim-audit-trail`.
**Реализация:** убрать проверку фичи.

### `AuditResource.java:74`
Журнал аудита на уровне окружения. Фича `apim-audit-trail`.
**Реализация:** убрать проверку фичи.

### `RoleScopeResource.java:80`
Создание пользовательских ролей. Фича `apim-custom-roles`.
**Реализация:** убрать проверку фичи.

### `ClientRegistrationProvidersResource.java:111`
Динамическая регистрация клиентов (DCR). Фича `apim-dcr-registration`.
**Реализация:** убрать проверку фичи.

### `KafkaExplorerResource.java:100`, `:124`, `:181`, `:213`, `:245`, `:303`, `:335`, `:388`
Все эндпоинты Kafka Explorer. Фича `apim-native-kafka-explorer`.
**Реализация:** во всех 8 точках убрать проверку фичи (или отключить в `GraviteeLicenseDomainService:32`).

### `AbstractResource.java:189` (–191)
Helper `isFeatureEnabled` для v1-ресурсов, которым нужна проверка внутри метода, а не
через аннотацию. Также платформенная лицензия.
**Реализация:** всегда возвращать `true`.

### `IdentityProvidersResource.java:137`
Проверка только при создании провайдера типа `OIDC`. Фича `apim-openid-connect-sso`.
**Реализация:** убрать проверку фичи внутри ветки `OIDC`.

### `GraviteeLicenseDomainService.java:32`
Точка над `getPlatformLicense().isFeatureEnabled(feature)` для core-слоя.
**Реализация:** всегда возвращать `true` из `isFeatureEnabled`.

### `GetConsoleCustomizationUseCase.java:55`
Без фичи консоль получает пустую кастомизацию, а не ошибку. Фича `oem-customization`.
**Реализация:** убрать проверку фичи.

### `LicenseDomainService.java:42` (–45)
Федерация проверяется не по фиче, а по тиру: разрешена при любой лицензии, кроме `oss`.
Отсутствие лицензии тоже считается разрешением.
**Реализация:** расширить условие, чтобы разрешало и `oss` (или всегда `true`).

### `LicenseDomainService.java:54` (–55)
Управляющая сторона той же фичи `apim-api-products`, что и `ApiProductManagerImpl:89`.
**Реализация:** всегда возвращать `true` из `isApiProductsFeatureEnabled()`.

---

<a id="04"></a>
## 04 · Флаг plugin.deployed()

Эти точки читают `deployed()` из манифеста плагина и кладут фабрику либо в `factories`,
либо в `notDeployedPluginFactories`. Лицензия накладывается поверх только в Management API
(слой 02). Важно: неза лицензированная фабрика не удаляется, а доступна через
`getFactoryById(id, includeNotDeployed = true)` — иначе консоль не получит схемы конфигурации.

### `DefaultEntrypointConnectorPluginManager.java:66`
При регистрации фабрика попадает в `factories` или `notDeployedPluginFactories`.
**Реализация:** класть всегда в `factories` (игнорировать `deployed()`).

### `DefaultEndpointConnectorPluginManager.java:66`
То же для endpoint-коннекторов.
**Реализация:** аналогично.

### `DefaultApiServicePluginManager.java:68`
То же для api-service-плагинов (health-check, dynamic properties, service discovery).
**Реализация:** аналогично.

### `DefaultEntrypointConnectorPlugin.java:84`
Проброс `deployed()` из делегата.
**Реализация:** `deployed()` возвращает `true`.

### `DefaultEndpointConnectorPlugin.java:83`
Проброс `deployed()`.
**Реализация:** `deployed()` возвращает `true`.

### `DefaultApiServicePlugin.java:79`
Проброс `deployed()`.
**Реализация:** `deployed()` возвращает `true`.

### `DefaultReactorPlugin.java:68`
Проброс `deployed()` (reactor).
**Реализация:** `deployed()` возвращает `true`.

### `GammaModulePlugin.java:69`
Единственное место в gravitee-gamma, связанное с флагом. Собственных проверок лицензии
не содержит.
**Реализация:** `deployed()` возвращает `true` (для единообразия).

### `AbstractPluginService.java:114`
Переносит `deployed()` и `manifest().feature()` в entity. Лицензия применяется выше (слой 02).
**Реализация:** в entity форсировать `deployed = true`.

### `AbstractConnectorPluginService.java:79`
То же для коннекторов.
**Реализация:** форсировать `deployed = true`.

### `PolicyPluginServiceImpl.java:96`
То же для policy-плагинов v4.
**Реализация:** форсировать `deployed = true`.

### `PolicyServiceImpl.java:144`
То же для policy-плагинов v1.
**Реализация:** форсировать `deployed = true`.

### `ApiServicePluginServiceImpl.java:71`
То же для api-service-плагинов.
**Реализация:** форсировать `deployed = true`.

### `ConnectorServiceImpl.java:61`
То же для коннекторов v1.
**Реализация:** форсировать `deployed = true`.

### `FetcherServiceImpl.java:83`
То же для fetcher-плагинов.
**Реализация:** форсировать `deployed = true`.

---

<a id="05"></a>
## 05 · Источник лицензии

Не проверки, а места, где лицензия попадает в `LicenseManager`. Если здесь ничего не
регистрировать, проверки (слои 01–04) видят «нет лицензии» и срабатывают.

### `LicenseDeployer.java:41`
`registerOrganizationLicense` на шлюзе: лицензия приезжает через sync вместе с остальными
событиями деплоя.
**Реализация:** не регистрировать организационную лицензию (или регистрировать «пустую»,
если логика завязана на наличие объекта).

### `SyncManager.java:172`
Тот же `registerOrganizationLicense` в Management API, из `LicenseRepository` по критерию
`ORGANIZATION`.
**Реализация:** не вызывать регистрацию из репозитория.

### `OrganizationCommandHandler.java:123`
Лицензия из Cockpit — сохраняется через `createOrUpdateOrganizationLicense`.
**Реализация:** не обрабатывать лицензионную часть команды (или не сохранять).

### `GraviteeLicenseResource.java:47`
Загрузка платформенной лицензии для консоли.
**Реализация:** не принимать/не сохранять платформенную лицензию.

### `OrganizationResource.java:72`
Загрузка организационной лицензии с откатом на платформенную.
**Реализация:** не принимать/не сохранять организационную лицензию.

---

## Самые центральные «рубильники»

| Что отключает | Точка |
|---------------|-------|
| Валидация плагинов по всему API | `ApiLicenseServiceImpl.java:58` |
| Проверка фич по платформе (core) | `GraviteeLicenseDomainService.java:32` |
| JAX-RS фильтр по аннотациям | `GraviteeLicenseFilter.java:46` |
| Флаг `deployed()` для всех плагинов | `PluginFilterByLicenseDomainService.java:37` + слой 04 (`deployed()` → `true`) |

## Заметки

- `PoliciesResource:136` и `ResourcesResource:113` дублируют логику
  `PluginFilterByLicenseDomainService:37` вне domain service.
- v1-ресурсы берут организационную лицензию, а `AbstractResource:189` — платформенную.
- `EndpointPluginQueryServiceLegacyWrapper:21` импортирует `LicenseManager`, но не использует.
- `gravitee-gamma` собственных проверок лицензии в `src/main` не содержит.
- Флаг `plugin.deployed()` вычисляется в `gravitee-plugin-core` (вне репозитория, в `.m2`):
  менять удобнее всего на уровне положения фабрик (слой 04), а не в самом `deployed()`.
