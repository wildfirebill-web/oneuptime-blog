# How to Migrate from Databricks to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Databricks, Migration, Analytics, Delta Lake, OLAP

Description: Move your analytical workloads from Databricks to ClickHouse by exporting Delta tables as Parquet and translating Spark SQL to ClickHouse SQL.

---

Databricks provides a powerful Spark-based platform, but for teams running primarily SQL analytics over fixed datasets, ClickHouse offers lower latency and lower cost. This guide covers exporting Delta tables and migrating queries.

## Choosing an Export Format

ClickHouse reads Parquet natively, making it the best export format from Databricks:

- Parquet - recommended, efficient, schema-aware
- CSV - simple but loses type information
- Delta - readable via ClickHouse DeltaLake engine (for S3-backed tables)

## Option 1 - Use the ClickHouse DeltaLake Engine

If your Delta table lives in S3, ClickHouse can read it directly:

```sql
SELECT *
FROM deltaLake('s3://my-bucket/delta/events/', 'ACCESS_KEY', 'SECRET_KEY')
LIMIT 10;
```

To copy into a local MergeTree table:

```sql
INSERT INTO events
SELECT *
FROM deltaLake('s3://my-bucket/delta/events/', 'ACCESS_KEY', 'SECRET_KEY');
```

## Option 2 - Export from Databricks as Parquet

In a Databricks notebook:

```python
df = spark.table("analytics.events")

df.write.mode("overwrite") \
  .parquet("s3://export-bucket/events_parquet/")
```

Then load into ClickHouse from S3:

```sql
INSERT INTO events
SELECT *
FROM s3(
    'https://s3.amazonaws.com/export-bucket/events_parquet/*.parquet',
    'ACCESS_KEY',
    'SECRET_KEY',
    'Parquet'
);
```

## Translating the Schema

Databricks Delta table schema (from `DESCRIBE TABLE`):

```text
event_id   BIGINT
user_id    BIGINT
event_type STRING
created_at TIMESTAMP
revenue    DOUBLE
```

ClickHouse DDL:

```sql
CREATE TABLE events (
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    revenue    Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## Query Translation

Databricks/Spark SQL uses similar syntax but with some differences:

Databricks:

```sql
SELECT
    date_trunc('HOUR', created_at) AS hour,
    event_type,
    count(1) AS cnt,
    percentile_approx(revenue, 0.95) AS p95
FROM events
WHERE created_at >= date_sub(current_timestamp(), 7)
GROUP BY 1, 2
ORDER BY cnt DESC
LIMIT 20;
```

ClickHouse equivalent:

```sql
SELECT
    toStartOfHour(created_at) AS hour,
    event_type,
    count() AS cnt,
    quantile(0.95)(revenue) AS p95
FROM events
WHERE created_at >= now() - INTERVAL 7 DAY
GROUP BY hour, event_type
ORDER BY cnt DESC
LIMIT 20;
```

## Spark SQL to ClickHouse Function Map

| Spark SQL | ClickHouse |
|---|---|
| `date_trunc('HOUR', ts)` | `toStartOfHour(ts)` |
| `current_timestamp()` | `now()` |
| `date_sub(ts, 7)` | `ts - INTERVAL 7 DAY` |
| `percentile_approx(col, 0.95)` | `quantile(0.95)(col)` |
| `size(array_col)` | `length(array_col)` |
| `explode(array_col)` | `ARRAY JOIN array_col` |

## Summary

Migrating from Databricks to ClickHouse is cleanest through the DeltaLake table engine when your data is in S3, or through Parquet export otherwise. The analytical SQL dialects are very similar, and most queries translate with minor function name adjustments.
