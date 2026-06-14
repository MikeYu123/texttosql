# Сценарии интеграции

Этот файл дополняет [основную спецификацию](../spec.md) и фиксирует последовательности взаимодействия T2SQL-service с внешними компонентами.

## Последовательность: настройка администратором

```mermaid
sequenceDiagram
    actor Admin as Администратор
    participant Console as Admin Console
    participant IdP as Identity Provider
    participant dbt as dbt Platform/Core
    participant DWH as Data Warehouse
    participant PolicyDB as Policy DB
    participant T2SQL as T2SQL-service
    participant Metadata as Metadata Store

    Admin->>Console: Создает workspace
    Console->>IdP: Подключает SSO и группы
    Console->>dbt: Подключает dbt project/environment
    Console->>DWH: Проверяет read-only connection
    Admin->>PolicyDB: Настраивает роли, лимиты, PII rules
    Console->>T2SQL: Запускает artifact import
    T2SQL->>dbt: Читает dbt artifacts
    T2SQL->>Metadata: Сохраняет normalized catalog
    T2SQL-->>Console: Семантика загружена
```

## Последовательность: публикация semantic changes

```mermaid
sequenceDiagram
    actor Steward as Data Steward / Analytics Engineer
    participant StewardUI as Steward UI
    participant Git as Git repo
    participant CI as CI/CD
    participant dbt as dbt build/test
    participant T2SQL as T2SQL-service
    participant Metadata as Metadata Store
    participant Synonyms as Synonym Dictionaries
    participant VectorDB as Vector DB

    Steward->>StewardUI: Изменяет dbt YAML/models/tests/glossary/synonyms
    StewardUI->>Git: Создает branch и automated commit
    StewardUI->>Git: Открывает PR
    Git->>CI: PR event
    CI->>dbt: dbt parse/build/test
    CI->>CI: semantic validation
    Steward->>Git: Review generated diff
    Git->>CI: Merge to main
    CI->>dbt: Deploy
    dbt->>T2SQL: Publish artifacts
    T2SQL->>Metadata: Normalize and store catalog
    T2SQL->>Synonyms: Load glossary aliases
    T2SQL->>VectorDB: Rebuild embeddings
```

## Последовательность: успешный metric query

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant UI as Chat UI
    participant T2SQL as T2SQL-service
    participant Metadata as Metadata Store
    participant Synonyms as Synonym Dictionaries
    participant VectorDB as Vector DB
    participant PolicyDB as Policy DB
    participant SL as dbt Semantic Layer
    participant DWH as Data Warehouse
    participant Audit as Audit Store

    User->>UI: "Выручка по странам за прошлый квартал"
    UI->>T2SQL: question + user context
    T2SQL->>PolicyDB: load roles and allowed domains
    T2SQL->>Metadata: lexical lookup revenue, country, period
    T2SQL->>Synonyms: expand revenue/country aliases
    T2SQL->>Metadata: lookup expanded terms
    T2SQL->>VectorDB: search similar objects and questions
    T2SQL->>PolicyDB: filter candidates by access
    T2SQL->>T2SQL: build semantic_layer query plan
    T2SQL->>PolicyDB: validate final plan
    T2SQL->>SL: execute metric query
    SL->>DWH: compiled SQL
    DWH-->>SL: result set
    SL-->>T2SQL: result + SQL + metadata
    T2SQL->>Audit: log trace
    T2SQL-->>UI: result, SQL, definitions, lineage
```

## Последовательность: неоднозначный вопрос

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant UI as Chat UI
    participant T2SQL as T2SQL-service
    participant Metadata as Metadata Store
    participant Synonyms as Synonym Dictionaries
    participant VectorDB as Vector DB

    User->>UI: "Покажи продажи по регионам"
    UI->>T2SQL: question
    T2SQL->>Metadata: lexical lookup sales, region
    T2SQL->>Synonyms: expand sales and region
    T2SQL->>VectorDB: search similar questions
    T2SQL->>T2SQL: detect ambiguity
    T2SQL-->>UI: "Что считать продажами и какой регион использовать?"
    User->>UI: "Выручку по региону клиента"
    UI->>T2SQL: clarification
    T2SQL-->>UI: continues metric query flow
```

## Последовательность: отказ по policy

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant UI as Chat UI
    participant T2SQL as T2SQL-service
    participant Metadata as Metadata Store
    participant PolicyDB as Policy DB
    participant Audit as Audit Store

    User->>UI: "Покажи email клиентов и их выручку"
    UI->>T2SQL: question + role
    T2SQL->>Metadata: resolve customer_email, revenue
    Metadata-->>T2SQL: customer_email tagged pii.direct_identifier
    T2SQL->>PolicyDB: check policy
    PolicyDB-->>T2SQL: deny
    T2SQL->>Audit: log denied request
    T2SQL-->>UI: "Нет доступа к email. Можно показать выручку по сегменту или стране."
```

## Последовательность: feedback → улучшение семантики

```mermaid
sequenceDiagram
    actor User as Пользователь
    actor Steward as Data Steward / Analytics Engineer
    participant UI as Chat UI
    participant StewardUI as Steward UI
    participant T2SQL as T2SQL-service
    participant Audit as Audit Store
    participant Git as Git repo
    participant CI as CI/CD
    participant Metadata as Metadata Store
    participant VectorDB as Vector DB

    User->>UI: Ставит feedback "неверная метрика"
    UI->>T2SQL: save feedback
    T2SQL->>Audit: attach feedback to question trace
    Steward->>StewardUI: review feedback queue
    StewardUI->>Audit: load question trace, SQL hash, selected objects
    StewardUI->>Git: automated commit with synonym / dbt definition / regression case
    Git->>CI: PR event
    Git->>CI: run validation
    CI-->>Git: pass
    Git->>Metadata: publish updated artifacts
    Metadata->>VectorDB: refresh embeddings
```

## Deployment

```mermaid
flowchart LR
    subgraph Client
        Web[Web UI]
        APIClient[API Client / BI / Slack]
        StewardUI[Steward UI]
    end

    subgraph AppRuntime
        Gateway[API Gateway]
        T2SQL[T2SQL-service]
    end

    subgraph Stores
        AppDB[(App DB / Session Store)]
        Metadata[(Metadata Store)]
        Synonyms[(Synonym Dictionaries)]
        VectorDB[(Vector DB)]
        PolicyDB[(Policy DB)]
        AuditDB[(Audit DB)]
    end

    subgraph dbtRuntime
        dbtJob[dbt Jobs]
        SemanticLayer[dbt Semantic Layer]
        MetricFlow[MetricFlow]
    end

    subgraph Data
        DWH[(Warehouse)]
        Cache[(Exports / Cache Tables)]
    end

    Web --> Gateway
    APIClient --> Gateway
    StewardUI --> Gateway
    Gateway --> T2SQL
    T2SQL --> AppDB
    T2SQL --> Metadata
    T2SQL --> Synonyms
    T2SQL --> VectorDB
    T2SQL --> PolicyDB
    T2SQL --> SemanticLayer
    T2SQL --> MetricFlow
    T2SQL --> DWH
    SemanticLayer --> DWH
    MetricFlow --> DWH
    dbtJob --> Metadata
    dbtJob --> Cache
    T2SQL --> AuditDB
```
