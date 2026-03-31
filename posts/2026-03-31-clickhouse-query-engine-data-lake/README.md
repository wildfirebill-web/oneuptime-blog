# How to Use ClickHouse as a Query Engine for Data Lakes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Lake, S3, Parquet, Query Engine, Analytics

Description: Use ClickHouse as a high-performance SQL query engine over your data lake, querying Parquet, ORC, and JSON files stored in S3 or compatible object storage.

---

## Overview

A query engine for data lakes runs SQL queries directly against files in object storage without requiring a separate ETL step to load data into a database. ClickHouse excels at this role due to its vectorized execution engine, Parquet columnar reader, and S3 integration.

## Supported Formats

ClickHouse can query the following file formats from object storage:

```text
Parquet, ORC, Arrow, Avro, JSON, JSONEachRow, CSV, TSV
```

Use the format that suits your data. Parquet and ORC are recommended for large analytical workloads because they support column pruning and predicate pushdown.

## Query Multiple Files with Globs

Use wildcards to query an entire data lake prefix:

```sql
SELECT
    toDate(event_ts)  AS day,
    event_type,
    count()           AS events
FROM s3(
    's3://datalake/events/year=2026/month=*/day=*/*.parquet',
    'KEY', 'SECRET', 'Parquet'
)
GROUP BY day, event_type
ORDER BY day, events DESC;
```

## Build a Logical Schema with External Tables

Create persistent external tables to avoid repeating connection details:

```sql
CREATE TABLE lake_events
ENGINE = S3(
    's3://datalake/events/**/*.parquet',
    'KEY', 'SECRET', 'Parquet'
)
SETTINGS s3_truncate_on_insert = 0;
```

Now query it like a normal table:

```sql
SELECT event_type, count()
FROM lake_events
WHERE event_ts >= today() - 7
GROUP BY event_type;
```

## Mix Local and Remote Tables in One Query

Join a local ClickHouse table against a data lake file:

```sql
SELECT
    u.name,
    count(e.event_id) AS total_events
FROM lake_events AS e
JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_ts >= today() - 30
GROUP BY u.name
ORDER BY total_events DESC
LIMIT 20;
```

`users` is a local MergeTree table, `lake_events` reads from S3.

## Use ClickHouse Local for Development

Test your queries against local Parquet files before running them against S3:

```bash
clickhouse-local --query "
    SELECT count()
    FROM file('/data/events/*.parquet', 'Parquet')
"
```

## Materialize Frequently Accessed Data

For hot data, import it into ClickHouse for faster queries:

```sql
INSERT INTO events_hot
SELECT *
FROM s3('s3://datalake/events/year=2026/**/*.parquet', 'KEY', 'SECRET', 'Parquet')
WHERE event_ts >= today() - 7;
```

## Partition Pruning

ClickHouse prunes partitions based on hive-style path components when you specify column types that match the path:

```sql
SELECT *
FROM s3(
    's3://datalake/events/year={year}/month={month}/*.parquet',
    'KEY', 'SECRET',
    'Parquet',
    'year UInt16, month UInt8, event_id String, event_ts DateTime'
)
WHERE year = 2026 AND month = 3;
```

## Summary

ClickHouse is a capable data lake query engine that queries Parquet, ORC, and JSON files in S3 without any data loading step. Use external tables for repeatable access, glob patterns for multi-file queries, and local MergeTree tables to accelerate queries on frequently accessed hot data.
