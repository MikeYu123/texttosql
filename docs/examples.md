# Примеры конфигураций и API

Этот файл дополняет [основную спецификацию](../spec.md) и содержит длинные примеры, которые не нужны в основном архитектурном документе.

## dbt project layout

```text
dbt_project/
  models/
    staging/
      sources.yml
      stg_orders.sql
      stg_customers.sql
      stg_products.sql

    marts/
      finance/
        fct_orders.sql
        dim_customers.sql
        dim_products.sql
        orders.yml
        customers.yml
        metrics.yml
        saved_queries.yml

  macros/
    security/
      mask_email.sql
      apply_region_filter.sql

  semantic/
    glossary.yml
    synonyms.yml
    t2sql_policies.yml
    regression_questions.yml
```

## Пример dbt semantic YAML

См. официальную документацию dbt по [semantic models](https://docs.getdbt.com/docs/build/semantic-models), [entities](https://docs.getdbt.com/docs/build/entities), [dimensions](https://docs.getdbt.com/docs/build/dimensions), [metrics](https://docs.getdbt.com/docs/build/metrics-overview) и [saved queries](https://docs.getdbt.com/docs/build/saved-queries).

```yaml
version: 2

models:
  - name: fct_orders
    description: "One row per order line."
    config:
      group: finance
      access: public
      contract:
        enforced: true
      meta:
        domain: finance
        owner: finance_analytics

    columns:
      - name: order_id
        data_type: string
        data_tests: [not_null]

      - name: customer_id
        data_type: string
        data_tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id

      - name: order_date
        data_type: date

      - name: net_revenue
        data_type: numeric

    semantic_model:
      name: orders
      defaults:
        agg_time_dimension: order_date

      entities:
        - name: order
          type: primary
          expr: order_id

        - name: customer
          type: foreign
          expr: customer_id

      dimensions:
        - name: order_date
          type: time
          expr: order_date
          type_params:
            time_granularity: day

        - name: order_status
          type: categorical
          expr: status

metrics:
  - name: revenue
    label: Revenue
    description: "Net revenue after discounts, before refunds."
    type: simple
    type_params:
      measure: net_revenue
    config:
      meta:
        domain: finance
        owner: finance_analytics
        certified: true
        synonyms:
          - sales
          - turnover
          - выручка

  - name: average_order_value
    label: Average Order Value
    type: ratio
    type_params:
      numerator: revenue
      denominator: orders
```

## Glossary и synonyms

```yaml
business_terms:
  - term: revenue
    canonical_object: metric.revenue
    definition: Net revenue after discounts, before refunds.
    owner: finance_analytics
    certified: true

synonyms:
  revenue:
    - sales
    - turnover
    - net sales
    - выручка
    - продажи

  customer__country:
    - country
    - client country
    - страна клиента
```

T2SQL-service использует этот справочник после первичного lookup в Metadata Store: расширяет пользовательские термины, повторно ищет canonical semantic objects и снижает неоднозначность.

## Policies

```yaml
defaults:
  allow_write_sql: false
  allow_raw_sources: false
  require_limit: true
  max_rows: 1000
  timeout_seconds: 60

roles:
  viewer:
    allowed_domains: [finance, sales]
    denied_tags: [pii, restricted]
    max_date_range_days: 730

  finance_analyst:
    allowed_domains: [finance]
    denied_tags: [direct_identifier]
    max_date_range_days: 3650

pii:
  email:
    action: mask
  phone:
    action: mask
  direct_identifier:
    action: deny
```

T2SQL-service проверяет policies дважды: сначала отсекает недоступные semantic objects до генерации query plan, затем валидирует итоговый semantic query или controlled SQL перед исполнением.

## API

### `POST /v1/questions`

```json
{
  "text": "Покажи выручку по странам за прошлый квартал",
  "domain": "finance",
  "session_id": "s_123",
  "options": {
    "show_sql": true,
    "max_rows": 100
  }
}
```

Response:

```json
{
  "question_id": "q_123",
  "status": "succeeded",
  "execution_mode": "semantic_layer",
  "data": [
    {"country": "Germany", "revenue": 1200000}
  ],
  "sql": "select ...",
  "semantic_objects": [
    {"type": "metric", "name": "revenue", "certified": true},
    {"type": "dimension", "name": "customer__country"}
  ],
  "explanation": {
    "metric": "Revenue = net revenue after discounts, before refunds.",
    "period": "previous quarter",
    "lineage": ["fct_orders", "dim_customers"]
  },
  "warnings": []
}
```

### `POST /v1/feedback`

```json
{
  "question_id": "q_123",
  "rating": "incorrect",
  "comment": "GMV был интерпретирован как revenue."
}
```

## Audit event

```json
{
  "event": "query_executed",
  "question_id": "q_123",
  "user_id": "u_123",
  "role": "finance_analyst",
  "semantic_version": "git_sha:abc123",
  "execution_mode": "semantic_layer",
  "semantic_objects": ["metric.revenue", "dimension.customer__country"],
  "sql_hash": "sha256:...",
  "row_count": 42,
  "latency_ms": 1840,
  "status": "succeeded",
  "created_at": "2026-06-14T12:00:00Z"
}
```

## Regression question

```yaml
id: revenue_by_country_previous_quarter
text: "Покажи выручку по странам за прошлый квартал"
expected:
  execution_mode: semantic_layer
  metrics:
    - revenue
  dimensions:
    - customer__country
  forbidden_columns:
    - customer_email
  require_certified_metric: true
```
