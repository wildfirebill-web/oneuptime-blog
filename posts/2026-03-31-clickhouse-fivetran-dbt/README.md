# How to Use ClickHouse with Fivetran and dbt

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Fivetran, dbt, ELT, Data Warehouse

Description: Connect Fivetran for automated data ingestion and dbt for SQL transformations to build a production ELT pipeline on ClickHouse.

---

## Why Fivetran and dbt Together

Fivetran handles the hard parts of data ingestion: managing API connectors, schema changes, incremental syncs, and retry logic. dbt handles transformation: SQL-based models, testing, documentation, and lineage. Together with ClickHouse as the warehouse, they form a complete ELT pipeline.

## Configuring Fivetran to Write to ClickHouse

Fivetran supports ClickHouse as a destination. In the Fivetran dashboard:

1. Go to Destinations and add a new destination
2. Select ClickHouse
3. Enter connection details:

```text
Host: ch.internal
Port: 8443  (HTTPS port)
Database: fivetran_raw
Username: fivetran_writer
Password: <secret>
SSL: Enabled
```

Fivetran creates schemas per source connector and tables matching the source schema.

## How Fivetran Writes Data

Fivetran uses an upsert approach. For ClickHouse, it inserts into a staging table and merges using `ReplacingMergeTree`. The destination tables include metadata columns:

```sql
-- Example Fivetran-managed table
SELECT _fivetran_synced, _fivetran_deleted, order_id, user_id, status
FROM fivetran_raw.stripe_charges
LIMIT 5;
```

## Staging Models in dbt

Layer staging models over Fivetran raw tables to clean and rename columns:

```sql
-- models/staging/stg_stripe_charges.sql
SELECT
    charge_id,
    customer_id,
    amount / 100.0 AS amount_usd,
    currency,
    status,
    toDateTime(created) AS created_at
FROM {{ source('stripe', 'charges') }}
WHERE NOT _fivetran_deleted
```

Define the source in `sources.yml`:

```text
sources:
  - name: stripe
    database: fivetran_raw
    tables:
      - name: charges
      - name: customers
      - name: refunds
```

## Mart Models for Business Analytics

```sql
-- models/marts/fct_revenue.sql
{{ config(materialized='incremental', order_by='(date, currency)') }}

SELECT
    toDate(created_at) AS date,
    currency,
    sum(amount_usd) AS gross_revenue,
    countIf(status = 'succeeded') AS successful_charges,
    countIf(status = 'failed') AS failed_charges
FROM {{ ref('stg_stripe_charges') }}
{% if is_incremental() %}
WHERE created_at >= (SELECT max(date) FROM {{ this }}) - 1
{% endif %}
GROUP BY date, currency
```

## Running dbt Tests

Add tests to ensure data quality after each Fivetran sync:

```text
models:
  - name: stg_stripe_charges
    columns:
      - name: charge_id
        tests:
          - not_null
          - unique
      - name: amount_usd
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

## Scheduling the Pipeline

Fivetran syncs run on their own schedule. Trigger dbt after each sync using Fivetran's dbt integration or a webhook to Airflow:

```bash
# Airflow task triggered by Fivetran webhook
dbt run --select tag:daily --target prod
dbt test --select tag:daily --target prod
```

## Summary

ClickHouse with Fivetran and dbt creates a production-grade ELT pipeline where Fivetran handles automated connector maintenance, ClickHouse provides fast analytical storage, and dbt delivers version-controlled SQL transformations with built-in testing.
