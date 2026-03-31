# How to Use ClickHouse as a Query Engine for Data Lakes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Lake, S3, Query Engine, Parquet, Delta Lake, Hudi, Iceberg

Description: Use ClickHouse as a high-performance query engine for data lakes on S3, supporting Parquet, Delta Lake, Apache Hudi, and Iceberg formats.

---

ClickHouse can function as a fast, cost-effective query engine for data lakes stored on S3, GCS, or Azure Blob Storage. Unlike dedicated compute engines, ClickHouse combines query engine and storage in one system, allowing you to query external lake data while also maintaining hot data locally.

## Supported Open Table Formats

ClickHouse supports the major open table formats:

- **Parquet** - Raw columnar files via `s3()` function
- **Delta Lake** - Via `deltaLake()` function and `DeltaLake` engine
- **Apache Hudi** - Via `hudi()` function and `Hudi` engine
- **Apache Iceberg** - Via `iceberg()` function and `Iceberg` engine

## Query Parquet from S3

```sql
SELECT
    event_type,
    count() AS events,
    sum(revenue) AS total_revenue
FROM s3(
    's3://my-data-lake/events/year=2025/month=01/*.parquet',
    'ACCESS_KEY', 'SECRET_KEY', 'Parquet'
)
GROUP BY event_type
ORDER BY total_revenue DESC
LIMIT 10;
```

## Query Iceberg Tables

```sql
SELECT *
FROM iceberg('s3://my-data-lake/iceberg-tables/sales/', 'ACCESS_KEY', 'SECRET_KEY')
WHERE order_date >= '2025-01-01'
LIMIT 100;
```

## Create External Table Definitions

Define external tables as persistent objects:

```sql
CREATE TABLE lake_events
ENGINE = S3('s3://my-data-lake/events/*.parquet', 'ACCESS_KEY', 'SECRET_KEY', 'Parquet');

CREATE TABLE lake_delta_orders
ENGINE = DeltaLake('s3://my-data-lake/delta/orders/', 'ACCESS_KEY', 'SECRET_KEY');
```

## Federation: Join Local and Lake Data

One of ClickHouse's strengths is joining local hot data with lake cold data:

```sql
SELECT
    l.event_type,
    count() AS total_events,
    sum(l.revenue) AS total_revenue,
    avg(c.lifetime_value) AS avg_customer_ltv
FROM lake_events l
JOIN customers c ON l.user_id = c.user_id
GROUP BY l.event_type
ORDER BY total_revenue DESC;
```

The `customers` table is stored locally in MergeTree, while `lake_events` is on S3.

## Accelerate Repeated Queries with Materialized Views

Cache S3 query results into local MergeTree tables:

```sql
CREATE TABLE events_hot (
    event_time DateTime,
    event_type LowCardinality(String),
    user_id UInt32,
    revenue Decimal(18, 4)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time);

-- Nightly refresh
INSERT INTO events_hot
SELECT event_time, event_type, user_id, revenue
FROM s3('s3://my-data-lake/events/year=2025/*.parquet', 'ACCESS_KEY', 'SECRET_KEY', 'Parquet');
```

## Configure S3 Credentials Globally

Avoid repeating credentials by configuring them in `config.xml`:

```xml
<named_collections>
    <data_lake>
        <access_key_id>ACCESS_KEY</access_key_id>
        <secret_access_key>SECRET_KEY</secret_access_key>
        <region>us-east-1</region>
    </data_lake>
</named_collections>
```

## Monitor Data Lake Query Performance

```sql
SELECT
    query,
    query_duration_ms,
    read_bytes,
    formatReadableSize(read_bytes) AS read_size
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%s3%'
ORDER BY query_duration_ms DESC
LIMIT 10;
```

## Summary

ClickHouse serves as a powerful query engine for data lakes, supporting Parquet, Delta Lake, Hudi, and Iceberg formats on S3. Use external tables for ad-hoc exploration, federation joins for combining lake and local data, and scheduled imports into MergeTree for performance-critical queries. This approach provides Presto/Trino-like capabilities with ClickHouse's superior query speed.
