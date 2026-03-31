# How to Optimize Schema for High-Frequency Inserts in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Insert, Schema Design, Performance, MergeTree, Optimization

Description: Learn how to design ClickHouse schemas that handle high-frequency inserts efficiently without sacrificing query performance.

---

High-frequency insert workloads - such as telemetry pipelines, clickstream ingestion, or log aggregation - require careful schema design in ClickHouse. The wrong schema can cause excessive part merges, memory pressure, and degraded query performance.

## Choose the Right Engine

For insert-heavy workloads, `MergeTree` and its variants are the standard choice. Avoid `ReplacingMergeTree` or `AggregatingMergeTree` unless you have a specific deduplication or pre-aggregation need, because they add overhead during merges.

```sql
CREATE TABLE events (
    event_time DateTime,
    event_type LowCardinality(String),
    user_id UInt64,
    session_id UUID,
    properties String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time, user_id);
```

## Use Appropriate Data Types

Over-sized data types waste memory and slow down inserts. Use the smallest type that fits your data:

```sql
-- Instead of:
user_id UInt64,
status String,

-- Prefer:
user_id UInt32,
status LowCardinality(String),
```

`LowCardinality` is particularly effective for string columns with fewer than 10,000 distinct values. It compresses significantly and speeds up GROUP BY operations.

## Minimize the Number of Columns

Each column adds overhead to the insert path. Combine related metadata into a JSON or Map column when those fields are rarely queried individually:

```sql
CREATE TABLE events (
    event_time DateTime,
    event_type LowCardinality(String),
    user_id UInt32,
    metadata Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time);
```

## Avoid Wide Primary Keys

Long ORDER BY clauses increase the cost of part sorting during inserts. Keep the primary key short and selective:

```sql
-- Good: short, selective key
ORDER BY (tenant_id, event_time)

-- Avoid: overly broad key
ORDER BY (tenant_id, event_type, source, region, event_time, user_id)
```

## Batch Inserts Correctly

Individual row inserts are very slow in ClickHouse. Always insert in batches of at least 1,000 rows, ideally 10,000-100,000:

```bash
# Using clickhouse-client
clickhouse-client --query="INSERT INTO events FORMAT JSONEachRow" < events_batch.jsonl
```

## Use Asynchronous Inserts

For clients that cannot batch, enable async inserts to let ClickHouse accumulate data:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;
```

Or configure it server-side per user:

```sql
ALTER USER ingest_user SETTINGS async_insert = 1, async_insert_busy_timeout_ms = 200;
```

## Tune Partition Granularity

Avoid too many partitions. Monthly partitions are usually better than daily for high-volume tables:

```sql
PARTITION BY toYYYYMM(event_time)
-- Instead of:
PARTITION BY toYYYYMMDD(event_time)
```

## Summary

Optimizing ClickHouse schemas for high-frequency inserts involves choosing the right engine, keeping primary keys short, using compact data types with `LowCardinality`, batching inserts, and avoiding over-partitioning. These practices reduce merge pressure and keep throughput high even under heavy write loads.
