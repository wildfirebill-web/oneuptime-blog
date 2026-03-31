# How to Use ClickHouse with Lightdash

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Lightdash, dbt, Business Intelligence, Analytics, Metric

Description: Connect Lightdash to ClickHouse through a dbt project to build open-source BI dashboards with metrics defined in version-controlled YAML.

---

Lightdash is an open-source BI tool that works on top of dbt. When combined with ClickHouse, it provides a modern analytics workflow where metrics are defined in code and dashboards are built from those definitions.

## Prerequisites

- A running ClickHouse instance
- dbt CLI installed (`pip install dbt-clickhouse`)
- Lightdash (self-hosted or cloud)

## Setting Up the dbt ClickHouse Project

Initialize a dbt project and configure the ClickHouse profile.

```bash
dbt init my_analytics
```

Edit `~/.dbt/profiles.yml`:

```text
my_analytics:
  target: dev
  outputs:
    dev:
      type: clickhouse
      schema: analytics
      host: localhost
      port: 8123
      user: default
      password: ""
      secure: false
```

## Creating dbt Models

Add a model `models/page_views_daily.sql`:

```sql
SELECT
    toDate(created_at) AS day,
    page,
    count() AS views,
    uniq(user_id) AS unique_users
FROM {{ source('raw', 'page_views') }}
GROUP BY day, page
```

## Defining Metrics in YAML

Create `models/page_views_daily.yml`:

```text
version: 2

models:
  - name: page_views_daily
    description: "Daily aggregated page views"
    columns:
      - name: day
        description: "Date of the events"
        meta:
          dimension:
            type: date
      - name: views
        meta:
          metrics:
            total_views:
              type: sum
      - name: unique_users
        meta:
          metrics:
            total_unique_users:
              type: sum
```

## Connecting Lightdash to ClickHouse via dbt

In Lightdash settings, add a new project:

1. Select "dbt CLI" or "dbt Cloud" as the project type.
2. Point Lightdash at your dbt project directory.
3. Choose ClickHouse as the warehouse and enter your connection credentials.
4. Click "Test and save."

Lightdash reads the dbt schema YAML and automatically generates an explore interface from your metric definitions.

## Building Dashboards

Once connected, users can:

- Browse tables defined in dbt models.
- Select dimensions like `day` and `page`.
- Select metrics like `total_views`.
- Save charts and add them to dashboards.

All without writing SQL directly.

## Running dbt and Refreshing Lightdash

```bash
dbt run --profiles-dir ~/.dbt
```

Lightdash picks up model changes on the next refresh or when you trigger a project compile.

## Summary

Lightdash bridges the gap between dbt's code-first approach and a self-service BI interface. Pairing it with ClickHouse gives your team fast query execution on large datasets, metrics defined in version-controlled YAML, and an open-source alternative to expensive BI platforms.
