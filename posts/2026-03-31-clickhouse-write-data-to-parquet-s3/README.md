# How to Write ClickHouse Data to Parquet on S3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Parquet, S3, Export, Data Lake

Description: Export ClickHouse table data to Parquet files on Amazon S3 for archival, sharing, or downstream data lake processing.

---

## Overview

Writing ClickHouse data to Parquet on S3 is useful for archiving old partitions, sharing data with other tools like Spark or Athena, and building data lake pipelines. ClickHouse supports this natively using the `s3` table function as an insert target.

## Export a Table to Parquet

Use `INSERT INTO FUNCTION s3(...)` to write directly to S3:

```sql
INSERT INTO FUNCTION s3(
    's3://my-bucket/exports/events_2026_01.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
)
SELECT *
FROM events
WHERE toYYYYMM(event_ts) = 202601;
```

## Write Partitioned Parquet Files

Write multiple files partitioned by date using a variable path:

```sql
INSERT INTO FUNCTION s3(
    's3://my-bucket/exports/events/year={year}/month={month}/data.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
)
SELECT
    *,
    toYear(event_ts)  AS year,
    toMonth(event_ts) AS month
FROM events
WHERE event_ts >= '2026-01-01';
```

ClickHouse creates separate files for each partition value combination.

## Compress the Output

Wrap the S3 URL with a compression codec:

```sql
INSERT INTO FUNCTION s3(
    's3://my-bucket/exports/events_2026.parquet.gz',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet',
    'gzip'
)
SELECT * FROM events;
```

Supported codecs include `gzip`, `brotli`, `zstd`, and `lz4`.

## Append to Existing Files with a Wildcard Path

Use a template path to write sharded output:

```sql
INSERT INTO FUNCTION s3(
    's3://my-bucket/exports/events_{_partition_id}.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
)
SELECT *
FROM events
WHERE event_ts >= today() - 7
SETTINGS s3_create_new_file_on_insert = 1;
```

## Verify the Export

Query the written files immediately to confirm correctness:

```sql
SELECT count()
FROM s3(
    's3://my-bucket/exports/events_2026_01.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);
```

## Use with AWS Athena

The Parquet files written by ClickHouse are fully compatible with AWS Athena. Register them in the Glue Data Catalog:

```sql
CREATE EXTERNAL TABLE events (
    event_id    string,
    user_id     bigint,
    event_type  string,
    event_ts    timestamp
)
STORED AS PARQUET
LOCATION 's3://my-bucket/exports/events/';
```

## Performance Tips

For large exports, increase the maximum parallel S3 upload threads:

```sql
SET max_threads = 16;
SET s3_min_upload_part_size = 33554432;
SET s3_upload_part_size = 67108864;
```

## Summary

ClickHouse writes Parquet files to S3 using the `s3` table function as an insert target. You can partition output by column values, compress with standard codecs, and consume the resulting files with any Parquet-compatible tool including Athena, Spark, and DuckDB.
