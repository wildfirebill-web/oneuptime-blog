# How to Benchmark Different Schema Designs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema Design, Benchmark, Performance, Testing, MergeTree

Description: Learn how to systematically benchmark different ClickHouse schema designs to find the optimal combination of ORDER BY key, partitioning, and column types for your workload.

---

Schema design choices in ClickHouse - ORDER BY key, partitioning, column types, and codecs - have a larger performance impact than in most databases. Systematic benchmarking helps you make evidence-based decisions before deploying to production.

## What to Benchmark

Key schema variables to test:

1. ORDER BY key composition and column order
2. Partition granularity (monthly vs. daily)
3. Column types (String vs. LowCardinality, UInt32 vs. UInt64)
4. Compression codecs (LZ4 vs. ZSTD, delta codecs)
5. Nullable vs. non-nullable columns

## Setting Up a Benchmark Environment

```sql
-- Load test dataset once into a raw table
CREATE TABLE raw_events (
    event_time DateTime,
    user_id UInt64,
    event_type String,
    country String,
    value Float64
) ENGINE = MergeTree()
ORDER BY event_time;

-- Fill with sample data (adjust to your actual volumes)
INSERT INTO raw_events
SELECT
    now() - rand() % 7776000,  -- random time in last 90 days
    rand() % 1000000,           -- 1M unique users
    ['purchase', 'view', 'click', 'signup'][rand() % 4 + 1],
    ['US', 'DE', 'JP', 'GB', 'CA'][rand() % 5 + 1],
    rand() / 4294967295 * 1000
FROM numbers(100000000);  -- 100M rows
```

## Schema Variant 1: Baseline

```sql
CREATE TABLE schema_v1 AS raw_events
ENGINE = MergeTree()
ORDER BY event_time;
INSERT INTO schema_v1 SELECT * FROM raw_events;
```

## Schema Variant 2: Event-Type First Key

```sql
CREATE TABLE schema_v2 (
    event_time DateTime,
    user_id UInt64,
    event_type LowCardinality(String),
    country LowCardinality(String),
    value Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, toDate(event_time), user_id);
INSERT INTO schema_v2 SELECT * FROM raw_events;
```

## Schema Variant 3: Columnar Optimization

```sql
CREATE TABLE schema_v3 (
    event_time DateTime CODEC(Delta, ZSTD(1)),
    user_id UInt32 CODEC(ZSTD(1)),
    event_type LowCardinality(String),
    country LowCardinality(String),
    value Float32 CODEC(ZSTD(1))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, country, toDate(event_time));
INSERT INTO schema_v3 SELECT * FROM raw_events;
```

## Running Benchmark Queries

```sql
-- Query 1: Time range filter
SELECT count() FROM schema_v1 WHERE event_time >= today() - 30;
SELECT count() FROM schema_v2 WHERE event_time >= today() - 30;
SELECT count() FROM schema_v3 WHERE event_time >= today() - 30;

-- Query 2: Event type aggregation
SELECT event_type, count(), avg(value) FROM schema_v1 GROUP BY event_type;
SELECT event_type, count(), avg(value) FROM schema_v2 GROUP BY event_type;
SELECT event_type, count(), avg(value) FROM schema_v3 GROUP BY event_type;

-- Query 3: Country + time
SELECT country, toDate(event_time), count()
FROM schema_v1
WHERE event_time >= today() - 7
GROUP BY country, toDate(event_time);
```

## Comparing Results

```sql
SELECT
    substr(query, 1, 60) AS query_short,
    query_duration_ms,
    read_rows,
    read_bytes,
    formatReadableSize(read_bytes) AS read_size,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%schema_v%'
  AND event_date = today()
ORDER BY query_short, query_duration_ms;
```

## Comparing Storage Size

```sql
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS compression_ratio
FROM system.parts
WHERE table LIKE 'schema_v%' AND active = 1
GROUP BY table
ORDER BY table;
```

## Summary

Benchmark ClickHouse schema designs by loading identical data into multiple schema variants, running your actual workload queries against each, and comparing query duration, marks read, memory usage, and storage size from `system.query_log` and `system.parts`. Focus on your top 5 most frequent queries when selecting the winning schema design.
