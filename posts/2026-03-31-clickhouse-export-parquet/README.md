# How to Export ClickHouse Data to Parquet Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Export, Parquet, Analytics, Data Lake

Description: Learn how to export ClickHouse data to Parquet files locally or directly to S3 using the Parquet format for efficient data lake storage.

---

Parquet is a columnar file format widely used in data lakes and analytics tools like Spark, Athena, and DuckDB. ClickHouse can write Parquet natively, making it easy to integrate with the broader data ecosystem.

## Exporting to Parquet with clickhouse-client

```bash
clickhouse-client \
  --host clickhouse \
  --user default \
  --password secret \
  --query "SELECT * FROM events WHERE toDate(ts) = today()" \
  --format Parquet \
  > events.parquet
```

## Exporting via HTTP Interface

```bash
curl -X GET \
  'http://clickhouse:8123/?query=SELECT+*+FROM+events&default_format=Parquet' \
  -H 'X-ClickHouse-User: default' \
  -H 'X-ClickHouse-Key: secret' \
  -o events.parquet
```

## Writing Directly to S3

The most production-ready approach is writing Parquet directly to S3:

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/my-data-lake/events/date=2026-03-31/part-000.parquet',
    'AWS_KEY',
    'AWS_SECRET',
    'Parquet'
)
SELECT * FROM events
WHERE toDate(ts) = '2026-03-31';
```

## Partitioned Export to S3

Export by month, creating a Hive-style partitioned layout:

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/my-data-lake/events/year=2026/month=03/data.parquet',
    'AWS_KEY', 'AWS_SECRET',
    'Parquet'
)
SELECT * FROM events
WHERE toYYYYMM(ts) = 202603;
```

## Automated Daily Export

Schedule a shell script with cron:

```bash
#!/bin/bash
DATE=$(date -d "yesterday" +%Y-%m-%d)
clickhouse-client \
  --query "SELECT * FROM events WHERE toDate(ts) = '${DATE}'" \
  --format Parquet \
  | aws s3 cp - "s3://my-data-lake/events/date=${DATE}/data.parquet"
```

## Verifying the Parquet Export

Read back with DuckDB:

```sql
-- In DuckDB
SELECT count(*), min(ts), max(ts)
FROM read_parquet('events.parquet');
```

## Summary

ClickHouse exports Parquet files locally via `--format Parquet` in clickhouse-client or directly to S3 using the `s3` table function. Use Hive-style partitioning (`date=`, `year=`) for compatibility with Athena and Spark, and automate daily exports with a cron job.
