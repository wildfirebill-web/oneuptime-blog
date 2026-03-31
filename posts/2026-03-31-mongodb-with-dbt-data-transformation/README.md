# How to Use MongoDB with dbt for Data Transformation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Dbt, Data Transformation, Analytics, ETL

Description: Learn how to use dbt with MongoDB via the dbt-mongodb adapter to define, test, and document data transformations directly on your MongoDB collections.

---

dbt (data build tool) brings software engineering practices to data transformations - version control, testing, documentation, and lineage. The `dbt-mongodb` adapter lets you run dbt models against MongoDB collections, transforming raw data into analytical views.

## Install the Adapter

```bash
pip install dbt-core dbt-mongodb
```

Verify the installation:

```bash
dbt --version
```

## Configure the Profile

Create or edit `~/.dbt/profiles.yml`:

```yaml
mongodb_project:
  target: dev
  outputs:
    dev:
      type: mongodb
      method: connection_string
      connection_string: "mongodb://localhost:27017/salesdb"
      database: salesdb
      schema: dbt_dev
    prod:
      type: mongodb
      method: connection_string
      connection_string: "{{ env_var('MONGODB_PROD_URI') }}"
      database: salesdb
      schema: dbt_prod
```

## Initialize a dbt Project

```bash
dbt init mongodb_project
cd mongodb_project
```

## Write a Source Model

Define your source collections in `models/sources.yml`:

```yaml
version: 2

sources:
  - name: raw
    database: salesdb
    schema: salesdb
    tables:
      - name: orders
      - name: customers
```

## Write a Transformation Model

Create `models/marts/order_summary.sql`:

```sql
{{ config(materialized='table') }}

SELECT
    category,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    MIN(createdAt) AS first_order_date,
    MAX(createdAt) AS last_order_date
FROM {{ source('raw', 'orders') }}
WHERE status = 'completed'
GROUP BY category
ORDER BY total_revenue DESC
```

For MongoDB, dbt-mongodb translates SQL models into aggregation pipelines.

## Add Schema Tests

Define tests in `models/marts/schema.yml`:

```yaml
version: 2

models:
  - name: order_summary
    columns:
      - name: category
        tests:
          - not_null
          - unique
      - name: total_revenue
        tests:
          - not_null
      - name: order_count
        tests:
          - not_null
```

## Run the Transformation

```bash
# Run all models
dbt run

# Run a specific model
dbt run --select order_summary

# Run tests
dbt test

# Generate and serve documentation
dbt docs generate
dbt docs serve
```

## Incremental Models

For large collections, use incremental models to process only new records:

```sql
{{ config(
    materialized='incremental',
    unique_key='_id'
) }}

SELECT *
FROM {{ source('raw', 'orders') }}
WHERE status = 'completed'

{% if is_incremental() %}
  AND createdAt > (SELECT MAX(createdAt) FROM {{ this }})
{% endif %}
```

## View Lineage

Run `dbt docs generate && dbt docs serve` to open a browser with an interactive lineage graph showing how each model depends on sources and other models.

## Summary

dbt with MongoDB brings SQL-based transformation models, automated testing, and documentation to MongoDB collections. Define sources, write SQL models that the adapter compiles to aggregation pipelines, add column-level tests, and use incremental materialization for large datasets. The result is a tested, documented, and version-controlled transformation layer on top of MongoDB.
