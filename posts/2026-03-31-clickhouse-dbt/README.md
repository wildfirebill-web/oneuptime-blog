# How to Use ClickHouse with dbt (Data Build Tool)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dbt, Analytics, Data Engineering, SQL, Transform

Description: Set up dbt with ClickHouse using dbt-clickhouse, write models with MergeTree engines, configure materializations, run incremental models, and test data quality.

---

dbt (data build tool) is the standard framework for SQL-based data transformation. The `dbt-clickhouse` adapter brings dbt's full model, test, and documentation workflow to ClickHouse. This guide covers installation, profile setup, materialization strategies, incremental models, and testing patterns.

## Installation

```bash
pip install dbt-clickhouse
```

Verify the installation:

```bash
dbt --version
```

## Initialize a dbt Project

```bash
dbt init analytics_project
cd analytics_project
```

## Profile Configuration

```yaml
# ~/.dbt/profiles.yml

analytics_project:
  target: dev
  outputs:
    dev:
      type: clickhouse
      host: localhost
      port: 8123
      schema: analytics_dbt
      user: default
      password: ""
      secure: false
      verify: false
      connect_timeout: 10
      send_receive_timeout: 300
      sync_request_timeout: 5
      compress_block_size: 1048576
      compression: ""
      driver: http
      check_exchange: true
      custom_settings:
        max_execution_time: 300
        max_memory_usage: 8000000000

    prod:
      type: clickhouse
      host: clickhouse.prod.internal
      port: 8443
      schema: analytics_dbt
      user: dbt_user
      password: "{{ env_var('CLICKHOUSE_PASSWORD') }}"
      secure: true
      verify: true
```

## Project Configuration

```yaml
# dbt_project.yml

name: analytics_project
version: '1.0.0'
config-version: 2

profile: analytics_project

model-paths: ["models"]
test-paths:  ["tests"]
seed-paths:  ["seeds"]
macro-paths: ["macros"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  analytics_project:
    staging:
      +materialized: view
      +engine: "MergeTree()"
    marts:
      +materialized: table
      +engine: "MergeTree()"
    incremental:
      +materialized: incremental
```

## Source Definitions

```yaml
# models/staging/sources.yml

version: 2

sources:
  - name: raw
    database: analytics
    schema: raw
    tables:
      - name: events
        description: "Raw event stream from application"
        columns:
          - name: event_id
            description: "Unique event identifier"
          - name: user_id
          - name: event_type
          - name: ts
            description: "Event timestamp at millisecond precision"
```

## Staging Model

```sql
-- models/staging/stg_events.sql

{{
  config(
    materialized = 'view'
  )
}}

SELECT
    event_id,
    user_id,
    session_id,
    lower(trim(event_type))                    AS event_type,
    page,
    JSONExtractString(properties, 'referrer')  AS referrer,
    JSONExtractString(properties, 'device')    AS device,
    ts
FROM {{ source('raw', 'events') }}
WHERE ts IS NOT NULL
  AND user_id > 0
  AND event_type != ''
```

## Mart Model with MergeTree Engine

```sql
-- models/marts/daily_event_stats.sql

{{
  config(
    materialized = 'table',
    engine       = 'MergeTree()',
    order_by     = '(event_date, event_type)',
    partition_by = 'toYYYYMM(event_date)'
  )
}}

SELECT
    toDate(ts)                AS event_date,
    event_type,
    count()                   AS total_events,
    uniq(user_id)             AS unique_users,
    uniq(session_id)          AS unique_sessions,
    countIf(device = 'mobile') AS mobile_events,
    countIf(device = 'desktop') AS desktop_events
FROM {{ ref('stg_events') }}
GROUP BY event_date, event_type
```

## Incremental Model

```sql
-- models/incremental/user_daily_activity.sql

{{
  config(
    materialized        = 'incremental',
    engine              = 'ReplacingMergeTree()',
    order_by            = '(activity_date, user_id)',
    partition_by        = 'toYYYYMM(activity_date)',
    unique_key          = ['activity_date', 'user_id'],
    incremental_strategy = 'delete+insert'
  )
}}

SELECT
    toDate(ts)        AS activity_date,
    user_id,
    count()           AS event_count,
    uniq(session_id)  AS session_count,
    min(ts)           AS first_event_at,
    max(ts)           AS last_event_at
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
WHERE ts >= (SELECT max(last_event_at) - INTERVAL 1 HOUR FROM {{ this }})
{% endif %}

GROUP BY activity_date, user_id
```

## Retention Cohort Model

```sql
-- models/marts/retention_cohorts.sql

{{
  config(
    materialized = 'table',
    engine       = 'MergeTree()',
    order_by     = '(cohort_week, week_number)'
  )
}}

WITH user_cohorts AS (
    SELECT
        user_id,
        toStartOfWeek(min(ts))  AS cohort_week,
        toStartOfWeek(ts)       AS activity_week
    FROM {{ ref('stg_events') }}
    WHERE ts >= today() - 90
    GROUP BY user_id, activity_week
)

SELECT
    cohort_week,
    dateDiff('week', cohort_week, activity_week) AS week_number,
    count(DISTINCT user_id)                       AS retained_users
FROM user_cohorts
GROUP BY cohort_week, week_number
ORDER BY cohort_week, week_number
```

## Session Attribution Model

```sql
-- models/marts/session_attribution.sql

{{
  config(
    materialized = 'table',
    engine       = 'MergeTree()',
    order_by     = '(session_date, session_id)'
  )
}}

SELECT
    session_id,
    toDate(min(ts))   AS session_date,
    min(user_id)      AS user_id,
    argMin(page, ts)  AS landing_page,
    argMax(page, ts)  AS exit_page,
    argMin(referrer, ts) AS referrer,
    dateDiff('second', min(ts), max(ts)) AS duration_seconds,
    count()           AS event_count
FROM {{ ref('stg_events') }}
WHERE session_id != ''
GROUP BY session_id
```

## Schema Tests

```yaml
# models/marts/schema.yml

version: 2

models:
  - name: daily_event_stats
    description: "Daily aggregated event statistics per event type"
    columns:
      - name: event_date
        tests:
          - not_null
      - name: event_type
        tests:
          - not_null
          - accepted_values:
              values: ['page_view', 'click', 'purchase', 'signup', 'login']
      - name: total_events
        tests:
          - not_null

  - name: user_daily_activity
    description: "Per-user daily activity summary"
    columns:
      - name: activity_date
        tests:
          - not_null
      - name: user_id
        tests:
          - not_null
      - name: event_count
        tests:
          - not_null
```

## Custom Singular Test

```sql
-- tests/assert_no_negative_event_counts.sql

SELECT *
FROM {{ ref('daily_event_stats') }}
WHERE total_events < 0
   OR unique_users < 0
```

## Macros

```sql
-- macros/clickhouse_date_trunc.sql

{% macro ch_date_trunc(unit, column) %}
  {% if unit == 'hour' %}
    toStartOfHour({{ column }})
  {% elif unit == 'day' %}
    toStartOfDay({{ column }})
  {% elif unit == 'week' %}
    toStartOfWeek({{ column }})
  {% elif unit == 'month' %}
    toStartOfMonth({{ column }})
  {% else %}
    {{ column }}
  {% endif %}
{% endmacro %}
```

Use in a model:

```sql
SELECT
    {{ ch_date_trunc('hour', 'ts') }} AS hour_bucket,
    count() AS events
FROM {{ ref('stg_events') }}
GROUP BY hour_bucket
```

## Running dbt

```bash
# Test connection
dbt debug

# Run all models
dbt run

# Run only marts
dbt run --select marts

# Run a specific model
dbt run --select daily_event_stats

# Run tests
dbt test

# Generate and serve docs
dbt docs generate
dbt docs serve
```

## Incremental Run

```bash
# Incremental models only process new data automatically
dbt run --select user_daily_activity

# Force a full refresh
dbt run --select user_daily_activity --full-refresh
```

## Summary

dbt-clickhouse brings all of dbt's transformation, testing, and documentation capabilities to ClickHouse. Use `view` materialization for staging models, `table` with an explicit `engine` for marts, and `incremental` with `delete+insert` strategy for large tables that receive daily updates. Combine `ReplacingMergeTree` with `unique_key` for idempotent incremental loads and add `not_null` and `accepted_values` tests to catch data quality issues before they reach dashboards.
