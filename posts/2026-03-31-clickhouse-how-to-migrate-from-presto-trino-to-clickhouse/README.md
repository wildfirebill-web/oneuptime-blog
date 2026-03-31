# How to Migrate from Presto/Trino to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Presto, Trino, Migration, Analytics, Query Engine

Description: Migrate from Presto or Trino to ClickHouse to move from a query-on-storage model to native columnar storage with faster aggregation performance.

---

Presto and Trino are federated SQL query engines that read from external storage (S3, HDFS, databases) at query time. ClickHouse stores data natively in a columnar format, which typically delivers 10-100x faster aggregation queries by eliminating the storage-query separation overhead.

## Architecture Difference

Presto/Trino pulls data from storage systems (S3, Hive, Iceberg) at query time. ClickHouse stores data locally in optimized columnar files and keeps indexes in memory. This means ClickHouse queries skip the data-fetch overhead entirely.

## Step 1 - Export Data from Presto/Trino

Use Trino CLI to export a table:

```bash
trino --server https://trino-coordinator:8080 \
  --catalog hive --schema analytics \
  --execute "SELECT * FROM page_views WHERE dt = '2024-01-15'" \
  --output-format TSV > page_views_20240115.tsv
```

For large tables, export to S3 directly via CREATE TABLE AS:

```sql
CREATE TABLE s3.exports.page_views_export
WITH (
  external_location = 's3://bucket/exports/page_views/',
  format = 'PARQUET'
) AS
SELECT * FROM hive.analytics.page_views;
```

## Step 2 - Create ClickHouse Table

Map Trino/Presto types to ClickHouse:

```sql
CREATE TABLE analytics.page_views
(
    dt           Date,
    page         LowCardinality(String),
    user_id      String,
    session_id   String,
    duration_ms  UInt32,
    country      LowCardinality(String),
    created_at   DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(dt)
ORDER BY (dt, page, user_id);
```

## Step 3 - Load from S3

```sql
INSERT INTO analytics.page_views
SELECT *
FROM s3(
    's3://bucket/exports/page_views/*.parquet',
    'AWS_KEY', 'AWS_SECRET',
    'Parquet'
);
```

## Step 4 - Translate SQL Syntax

Most standard SQL works in both systems. Key differences:

```sql
-- Trino: date arithmetic
SELECT date_add('day', -7, current_date)

-- ClickHouse
SELECT today() - 7
```

```sql
-- Trino: string functions
SELECT regexp_extract(url, 'https?://([^/]+)', 1) AS domain

-- ClickHouse
SELECT extract(url, 'https?://([^/]+)') AS domain
```

```sql
-- Trino: approximate distinct
SELECT approx_distinct(user_id) FROM page_views

-- ClickHouse
SELECT uniq(user_id) FROM analytics.page_views
```

## Step 5 - Handle Missing JOIN Performance

Trino is optimized for large distributed JOINs across catalogs. In ClickHouse, for better JOIN performance, denormalize frequently joined tables or use dictionaries:

```sql
CREATE DICTIONARY dim_pages (
    page_id UInt64,
    page_name String,
    category String
)
PRIMARY KEY page_id
SOURCE(CLICKHOUSE(TABLE 'dim_pages'))
LIFETIME(300)
LAYOUT(HASHED());
```

## Summary

Migrating from Presto/Trino to ClickHouse trades a flexible query-on-storage model for a self-contained columnar database with faster aggregation performance. The SQL syntax is very similar, and the main work involves re-exporting data into ClickHouse's native storage format and replacing a few Presto-specific functions.
