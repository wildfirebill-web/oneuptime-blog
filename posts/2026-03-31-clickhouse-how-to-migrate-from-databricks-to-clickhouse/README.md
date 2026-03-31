# How to Migrate from Databricks to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Databricks, Migration, Delta Lake, Analytics, Spark

Description: Migrate analytical workloads from Databricks to ClickHouse to reduce costs while maintaining fast query performance on large datasets stored in columnar format.

---

Databricks is a powerful unified analytics platform, but its compute-per-query pricing model becomes expensive for high-frequency or always-on analytical workloads. ClickHouse provides fast analytical queries on your own infrastructure at predictable cost.

## When to Consider This Migration

Move to ClickHouse when:
- You run frequent short analytical queries that spin up Databricks clusters
- Your data team needs sub-second dashboard queries
- Cloud compute costs for always-on Databricks clusters are high
- Your workload is primarily aggregation, not large-scale ML or ETL

## Step 1 - Export Data from Databricks Delta Tables

Use Databricks notebooks to write Delta tables to Parquet on S3:

```python
# In a Databricks notebook
df = spark.table("analytics.page_views")
df.write.mode("overwrite").parquet("s3://bucket/clickhouse-migration/page_views/")
```

Or export from Delta Lake directly:

```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "s3://bucket/delta/page_views")
delta_table.toDF().write.parquet("s3://bucket/export/page_views/")
```

## Step 2 - Create the ClickHouse Table

```sql
CREATE TABLE analytics.page_views
(
    event_id    String,
    user_id     String,
    page        LowCardinality(String),
    duration_ms UInt32 DEFAULT 0,
    event_date  Date,
    created_at  DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id, event_id);
```

## Step 3 - Load from S3

```sql
INSERT INTO analytics.page_views
SELECT *
FROM s3(
    's3://bucket/clickhouse-migration/page_views/*.parquet',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY',
    'Parquet'
);
```

## Step 4 - Translate Spark SQL to ClickHouse SQL

```sql
-- Spark SQL
SELECT
  date_trunc('hour', created_at) AS hour,
  count(*) AS events,
  approx_count_distinct(user_id) AS unique_users
FROM page_views
GROUP BY hour
ORDER BY hour

-- ClickHouse
SELECT
  toStartOfHour(created_at) AS hour,
  count() AS events,
  uniq(user_id) AS unique_users
FROM analytics.page_views
GROUP BY hour
ORDER BY hour
```

```sql
-- Spark SQL: percentile
SELECT percentile_approx(duration_ms, 0.95) AS p95 FROM page_views

-- ClickHouse
SELECT quantile(0.95)(duration_ms) AS p95 FROM analytics.page_views
```

## Step 5 - Replace Databricks Jobs with ClickHouse Materialized Views

Many Databricks jobs that pre-aggregate data can be replaced with ClickHouse materialized views:

```sql
CREATE MATERIALIZED VIEW analytics.hourly_stats
ENGINE = SummingMergeTree()
ORDER BY (hour, page)
AS
SELECT
    toStartOfHour(created_at) AS hour,
    page,
    count() AS events,
    sum(duration_ms) AS total_duration
FROM analytics.page_views
GROUP BY hour, page;
```

## Summary

Migrating from Databricks to ClickHouse for analytical queries reduces cloud costs by replacing on-demand cluster compute with efficient columnar storage. Data exports via Parquet, the SQL migration is mostly minor function renames, and materialized views replace batch aggregation jobs.
