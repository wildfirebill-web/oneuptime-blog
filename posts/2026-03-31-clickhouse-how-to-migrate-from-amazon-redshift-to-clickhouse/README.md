# How to Migrate from Amazon Redshift to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Amazon Redshift, Migration, Data Warehouse, Analytics, ETL

Description: Migrate your Amazon Redshift data warehouse to ClickHouse to reduce costs and improve query performance on large analytical datasets.

---

Amazon Redshift is a managed columnar data warehouse, but its cost model and query performance can become limiting as data grows. ClickHouse offers faster analytical queries, lower storage costs, and more flexible deployment options.

## Key Differences

| Feature | Redshift | ClickHouse |
|---------|----------|------------|
| Distribution model | Cluster nodes | Shards + replicas |
| Sort key | SORTKEY (one per table) | ORDER BY (primary index) |
| Distribution key | DISTKEY | Sharding key |
| Compression | Automatic per column | Automatic (LZ4/ZSTD) |
| Approximate functions | HLL sketch | uniq(), uniqHLL12() |

## Step 1 - Export Data from Redshift

Use Redshift UNLOAD to export to S3 in Parquet format:

```sql
UNLOAD ('SELECT * FROM analytics.page_views')
TO 's3://your-bucket/exports/page_views/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftS3Role'
FORMAT PARQUET
ALLOWOVERWRITE;
```

## Step 2 - Create the Destination Table in ClickHouse

Map Redshift types to ClickHouse equivalents:

```sql
CREATE TABLE analytics.page_views
(
    event_id     String,
    user_id      String,
    page         String,
    duration_ms  UInt32,
    created_at   DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id)
SETTINGS index_granularity = 8192;
```

## Step 3 - Load from S3 into ClickHouse

```sql
INSERT INTO analytics.page_views
SELECT *
FROM s3(
    's3://your-bucket/exports/page_views/*.parquet',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY',
    'Parquet'
);
```

## Step 4 - Migrate Redshift SQL Queries

Common syntax differences to update:

Redshift:
```sql
SELECT DATEADD(day, -7, GETDATE())
```

ClickHouse:
```sql
SELECT now() - INTERVAL 7 DAY
```

Redshift approximate count distinct:
```sql
SELECT APPROXIMATE COUNT(DISTINCT user_id) FROM page_views
```

ClickHouse:
```sql
SELECT uniq(user_id) FROM page_views
```

Redshift window functions work similarly but with minor syntax changes:
```sql
-- Redshift
SELECT user_id, ROW_NUMBER() OVER (PARTITION BY page ORDER BY created_at) AS rn
FROM page_views

-- ClickHouse
SELECT user_id, row_number() OVER (PARTITION BY page ORDER BY created_at) AS rn
FROM page_views
```

## Step 5 - Validate Row Counts and Aggregates

```sql
-- Run on both Redshift and ClickHouse to compare
SELECT
    toDate(created_at) AS day,
    count() AS events,
    uniq(user_id) AS unique_users
FROM analytics.page_views
GROUP BY day
ORDER BY day;
```

## Step 6 - Update Connection Strings

Replace the Redshift JDBC/ODBC connection with ClickHouse drivers in your application and BI tools.

## Summary

Migrating from Redshift to ClickHouse involves exporting data via UNLOAD to S3, loading it with ClickHouse's S3 table function, and translating query syntax differences. The result is typically faster query performance, lower storage costs, and more control over your deployment configuration.
