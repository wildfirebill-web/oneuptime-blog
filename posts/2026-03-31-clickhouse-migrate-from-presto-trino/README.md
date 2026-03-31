# How to Migrate from Presto/Trino to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Presto, Trino, Migration, Analytics, OLAP, Data Warehouse

Description: Learn how to move your ad-hoc analytics workloads from Presto or Trino to ClickHouse for faster queries and simpler infrastructure management.

---

Presto and Trino are excellent federated query engines, but they add operational complexity when the goal is fast analytics over a single large dataset. ClickHouse keeps data local, uses columnar storage natively, and consistently outperforms Presto/Trino on aggregation-heavy workloads.

## Architecture Differences

| Presto/Trino | ClickHouse |
|---|---|
| Query engine over external storage | Integrated storage and compute |
| Hive/Iceberg/S3 data sources | MergeTree-family engines |
| Coordinator + Worker nodes | Single node or sharded cluster |
| JVM-based | C++ native |

## Translating the Schema

Presto/Trino uses Hive-style DDL. A typical Hive Metastore table:

```sql
CREATE EXTERNAL TABLE web_logs (
    ts          BIGINT,
    path        VARCHAR,
    status_code INT,
    duration_ms DOUBLE
)
STORED AS PARQUET
LOCATION 's3://my-bucket/web_logs/';
```

ClickHouse equivalent:

```sql
CREATE TABLE web_logs (
    ts          DateTime,
    path        LowCardinality(String),
    status_code UInt16,
    duration_ms Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (path, ts);
```

## Exporting Data from Presto/Trino

Export to Parquet using the Trino CLI:

```bash
trino --execute "SELECT * FROM hive.mydb.web_logs" \
      --output-format TSV > web_logs.tsv
```

Or export directly to S3 in Parquet format:

```sql
CREATE TABLE hive.export.web_logs_export
WITH (format = 'PARQUET', external_location = 's3://export-bucket/web_logs/')
AS SELECT * FROM hive.mydb.web_logs;
```

## Loading Parquet into ClickHouse

ClickHouse can read Parquet files directly from S3:

```sql
INSERT INTO web_logs
SELECT
    fromUnixTimestamp(ts) AS ts,
    path,
    status_code,
    duration_ms
FROM s3(
    'https://s3.amazonaws.com/export-bucket/web_logs/*.parquet',
    'ACCESS_KEY', 'SECRET_KEY',
    'Parquet'
);
```

Or from a local file:

```bash
clickhouse-client \
  --query "INSERT INTO web_logs FORMAT Parquet" \
  < web_logs.parquet
```

## Translating Queries

Presto/Trino query:

```sql
SELECT
    date_trunc('hour', from_unixtime(ts)) AS hour,
    path,
    count(*) AS hits,
    avg(duration_ms) AS avg_ms
FROM web_logs
WHERE ts >= to_unixtime(now()) - 86400
GROUP BY 1, 2
ORDER BY hits DESC
LIMIT 20;
```

ClickHouse equivalent:

```sql
SELECT
    toStartOfHour(ts) AS hour,
    path,
    count() AS hits,
    avg(duration_ms) AS avg_ms
FROM web_logs
WHERE ts >= now() - INTERVAL 1 DAY
GROUP BY hour, path
ORDER BY hits DESC
LIMIT 20;
```

## Key Function Mappings

| Presto/Trino | ClickHouse |
|---|---|
| `date_trunc('hour', ts)` | `toStartOfHour(ts)` |
| `approx_distinct(col)` | `uniq(col)` |
| `approx_percentile(col, 0.95)` | `quantile(0.95)(col)` |
| `cardinality(array)` | `length(array)` |
| `element_at(map, key)` | `map[key]` |

## Summary

Migrating from Presto/Trino to ClickHouse simplifies your infrastructure by removing the dependency on a separate metastore and distributed query engine. Data exports via Parquet and S3 give a clean migration path, and most SQL translates with small function name changes.
