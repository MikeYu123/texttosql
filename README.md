# dbt Semantic Layer + Text-to-SQL

Документация по дизайну T2SQL-service и его интеграции с dbt Semantic Layer.

## Навигация

| Документ | Что внутри |
|---|---|
| [Спецификация](spec.md) | Scope, роли, архитектура, T2SQL-service, semantic layer, pipeline, security, CI/CD, MVP. |
| [Примеры конфигураций и API](docs/examples.md) | dbt project layout, semantic YAML, glossary, policies, API payloads, audit event, regression case. |
| [Сценарии интеграции](docs/integration-scenarios.md) | Onboarding, публикация semantic changes, успешный query flow, ambiguity, policy denial, feedback loop, deployment. |

## Рекомендуемый порядок чтения

1. [Спецификация](spec.md)
2. [Сценарии интеграции](docs/integration-scenarios.md)
3. [Примеры конфигураций и API](docs/examples.md)
