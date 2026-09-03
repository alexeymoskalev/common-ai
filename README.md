# common-ai

Общие материалы для AI-агентов: правила кодирования, определения субагентов, справочники по проектам.

## Структура

| Путь | Что это |
| --- | --- |
| `gravitee-apim-4/` | Правила и субагент для монорепозитория Gravitee API Management **4.12.14** |
| `license.md` | Разбор лицензионных гейтов Gravitee APIM 4.12.14 — где backend проверяет лицензию |

## `gravitee-apim-4/`

Пути внутри повторяют структуру монорепозитория Gravitee — папку можно распаковать поверх корня проекта.

```
.claude/agents/gravitee-4-backend.md   субагент: backend-задачи под APIM 4 (Vert.x 5, RxJava 3)
.agent-rules/
  gravitee-v4-architecture.md          карта модулей, v4 definition model, фазы флоу, build-гейты
  gravitee-v4-reactive-api.md          ExecutionContext, body/chunks/messages, interrupt, логирование
  gravitee-v4-plugins.md               policy / entrypoint / endpoint / api-service: анатомия и скелеты
  vertx.md                             Vert.x 5 + RxJava 3: шедулеры, event loop, core↔rxjava3
  gravitee-testing.md                  реактивные unit-тесты и gateway integration tests
AGENTS.md                              корневой AGENTS.md проекта с таблицей правил
gravitee-apim-rest-api/AGENTS.md       правила модуля rest-api (OpenAPI-first + гексагональное ядро)
```

Базовая версия, под которую всё выверено: APIM **4.12.14**, Java **21**, Vert.x **5.0.12**, `gravitee-gateway-api` **6.3.0**.
