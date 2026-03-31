# How to Use ClickHouse in a Modern Data Stack Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Modern Data Stack, dbt, Airbyte, Data Architecture

Description: Integrate ClickHouse into the modern data stack alongside Airbyte and dbt to build an open-source ELT pipeline with SQL-based transformations.

---

## What Is the Modern Data Stack

The modern data stack (MDS) is a collection of cloud-native tools that work together for data ingestion, storage, transformation, and analysis:

- **Ingestion**: Airbyte, Fivetran, or Stitch (extract and load raw data)
- **Storage**: ClickHouse (warehouse)
- **Transformation**: dbt (SQL-based models)
- **BI**: Grafana, Superset, or Metabase

ClickHouse replaces or complements cloud data warehouses like Snowflake or BigQuery in this stack.

## Why ClickHouse in the MDS

ClickHouse fits the MDS because:
- It supports SQL natively, making dbt a first-class citizen
- It ingests data at very high throughput for large ELT pipelines
- It is open source and can be self-hosted or used via ClickHouse Cloud
- Its columnar engine delivers sub-second query performance for BI tools

## Setting Up Airbyte to Load into ClickHouse

Airbyte has a native ClickHouse destination connector. Configure it with:

```text
Destination: ClickHouse
Host: ch.internal
Port: 8123
Database: raw
Username: airbyte_writer
Password: <secret>
SSL: false
```

Airbyte loads data into staging tables prefixed with `_airbyte_raw_`. Enable normalization to get clean destination tables.

## Configuring dbt for ClickHouse

Install the dbt-clickhouse adapter:

```bash
pip install dbt-clickhouse
```

Configure `profiles.yml`:

```text
clickhouse_project:
  target: prod
  outputs:
    prod:
      type: clickhouse
      host: ch.internal
      port: 8123
      user: dbt_user
      password: "{{ env_var('CH_PASSWORD') }}"
      schema: analytics
      secure: false
```

## A dbt Model on ClickHouse

```sql
-- models/marts/fct_daily_orders.sql
{{ config(
    materialized='incremental',
    engine='MergeTree()',
    order_by='(order_date, status)',
    partition_by='toYYYYMM(order_date)',
    unique_key='order_id'
) }}

SELECT
    order_id,
    toDate(created_at) AS order_date,
    user_id,
    status,
    total_cents / 100.0 AS total_usd
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
WHERE created_at >= (SELECT max(order_date) FROM {{ this }})
{% endif %}
```

## Running the dbt Pipeline

```bash
dbt run --target prod --select marts.fct_daily_orders
dbt test --target prod --select marts.fct_daily_orders
```

## Orchestrating with Airflow

Schedule the full ELT pipeline with Apache Airflow:

```text
airbyte_sync >> dbt_run_staging >> dbt_run_marts >> alert_on_failure
```

The DAG triggers an Airbyte sync, waits for completion, runs dbt staging models, then runs marts.

## Summary

ClickHouse integrates naturally into the modern data stack alongside Airbyte for ingestion and dbt for transformation, delivering a fully open-source ELT pipeline with SQL-based analytics at high performance and low cost.
