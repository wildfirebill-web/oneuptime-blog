# How to Read Parquet Files from S3 in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Parquet, S3, Data Lake, Query Optimization

Description: Query Parquet files stored in Amazon S3 directly from ClickHouse using the s3 table function and S3 table engine for data lake analytics.

---

## Overview

ClickHouse can query Parquet files stored in Amazon S3 without loading them into local storage. This enables data lake queries, ad-hoc analytics, and cost-effective storage for infrequently accessed data.

## Ad-Hoc Query with the s3 Table Function

Use the `s3` table function for quick, one-off queries:

```sql
SELECT
    region,
    count()      AS num_orders,
    sum(revenue) AS total_revenue
FROM s3(
    's3://my-bucket/sales/2026/**/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
)
GROUP BY region
ORDER BY total_revenue DESC;
```

Use glob patterns to query multiple files in one statement. ClickHouse reads only the columns referenced in the query thanks to Parquet's columnar format.

## Inspect Schema Before Querying

Use `DESCRIBE` to see the inferred schema:

```sql
DESCRIBE TABLE s3(
    's3://my-bucket/sales/2026/01/part-00000.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);
```

## Create a Persistent S3 Table

For repeated access, create a table backed by S3:

```sql
CREATE TABLE s3_sales
ENGINE = S3(
    's3://my-bucket/sales/2026/**/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);
```

Query it like a regular table:

```sql
SELECT product_id, sum(quantity)
FROM s3_sales
WHERE sale_date >= '2026-01-01'
GROUP BY product_id;
```

## Filter Pushdown with Hive Partitioning

If files are organized by date as `s3://my-bucket/sales/year=2026/month=03/`, enable hive partition pruning:

```sql
SELECT *
FROM s3(
    's3://my-bucket/sales/year=*/month=*/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet',
    'year UInt16, month UInt8, region String, revenue Float64'
)
WHERE year = 2026 AND month = 3;
```

ClickHouse prunes files based on partition path components.

## Load Parquet Data into ClickHouse

To persist query results locally for fast repeated access:

```sql
INSERT INTO local_sales
SELECT *
FROM s3(
    's3://my-bucket/sales/2026/**/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);
```

## Use IAM Role Instead of Keys

On EC2 or ECS, use instance profile credentials:

```sql
SELECT count()
FROM s3(
    's3://my-bucket/sales/*.parquet',
    'Parquet'
);
```

ClickHouse automatically picks up IAM role credentials when no keys are specified.

## Tune S3 Read Performance

Increase parallel S3 reads:

```sql
SET max_threads = 16;
SET max_download_threads = 8;
SET max_download_buffer_size = 10485760;
```

## Summary

ClickHouse reads Parquet files from S3 natively using the `s3` table function or S3 table engine. Use glob patterns for multi-file queries, enable hive partitioning for automatic pruning, and tune download threads for maximum throughput.
