# How to Export ClickHouse Data to S3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, S3, Export, Data Lake, AWS

Description: Learn how to export ClickHouse tables and query results to Amazon S3 in CSV, Parquet, and JSON formats using the s3 table function.

---

ClickHouse's `s3` table function lets you write query results directly to Amazon S3 without intermediate files, enabling efficient data lake exports and backups.

## Basic Export to S3

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/my-bucket/exports/events.csv',
    'YOUR_AWS_ACCESS_KEY',
    'YOUR_AWS_SECRET_KEY',
    'CSVWithNames'
)
SELECT * FROM events
WHERE toDate(ts) = today() - 1;
```

## Using IAM Roles (No Credentials in Query)

On EC2 or ECS with an attached IAM role, omit credentials:

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/my-bucket/exports/events.parquet',
    'Parquet'
)
SELECT * FROM events
WHERE toDate(ts) = today() - 1;
```

Configure `use_environment_credentials = 1` in `config.xml`:

```xml
<s3>
  <use_environment_credentials>1</use_environment_credentials>
</s3>
```

## Exporting Multiple Files with Globs

Write sharded output files:

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/my-bucket/events/part_{_part_index}.parquet',
    'Parquet'
)
SELECT * FROM events;
```

## Partitioned Export by Date

```bash
#!/bin/bash
for date in $(seq -f "%04g-%02g-%02g" 2026 01 01 to 2026 03 31); do
  clickhouse-client --query "
    INSERT INTO FUNCTION s3(
      's3://my-bucket/events/date=${date}/data.parquet',
      'Parquet'
    )
    SELECT * FROM events WHERE toDate(ts) = '${date}'
  "
done
```

## Reading Back from S3

Verify the export:

```sql
SELECT count(), min(ts), max(ts)
FROM s3(
    'https://s3.amazonaws.com/my-bucket/exports/events.parquet',
    'Parquet'
);
```

## Export to S3-Compatible Storage (MinIO)

```sql
INSERT INTO FUNCTION s3(
    'http://minio:9000/my-bucket/events.csv',
    'minio_user', 'minio_password',
    'CSVWithNames'
)
SELECT * FROM events LIMIT 10000;
```

## Summary

ClickHouse exports to S3 using `INSERT INTO FUNCTION s3(url, [key, secret,] format)`. Use IAM role-based authentication for production, write Parquet for downstream Spark/Athena compatibility, and use glob patterns to shard large exports into multiple files.
