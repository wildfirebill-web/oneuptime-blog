# How to Handle High-Frequency Streaming Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Streaming, Kafka, High Frequency, Insert, Buffer Table

Description: Practical guide to ingesting high-frequency streaming data into ClickHouse using async inserts, Buffer tables, and Kafka engine for stable throughput.

---

## The Challenge of High-Frequency Inserts

ClickHouse stores data in parts on disk. Each INSERT creates at least one part. If you insert thousands of small batches per second, you'll create too many parts, triggering the "Too many parts" error and stalling merges. The goal is to batch inserts effectively.

## Strategy 1 - Async Inserts

ClickHouse 22.x+ supports async inserts, which buffer small writes server-side before flushing to storage.

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;
SET async_insert_max_data_size = 10000000;  -- 10 MB
SET async_insert_busy_timeout_ms = 200;
```

With async inserts, individual row inserts are safe:

```bash
clickhouse-client --async_insert=1 --wait_for_async_insert=0 \
  --query="INSERT INTO events FORMAT JSONEachRow" < event.json
```

## Strategy 2 - Buffer Tables

Buffer tables accumulate writes in memory and flush to the target table in batches:

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(
    currentDatabase(), events,
    16,          -- num_layers
    10, 100,     -- min/max flush seconds
    10000, 1000000, -- min/max flush rows
    10000000, 100000000  -- min/max flush bytes
);
```

Applications write to `events_buffer` and ClickHouse handles the batching automatically.

## Strategy 3 - Kafka Engine

For Kafka-based streaming, use the Kafka table engine:

```sql
CREATE TABLE events_kafka (
    event_type String,
    user_id    UInt64,
    ts         DateTime,
    payload    String
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'events',
    kafka_group_name  = 'clickhouse_consumer',
    kafka_format      = 'JSONEachRow';

CREATE MATERIALIZED VIEW events_mv TO events AS
SELECT event_type, user_id, ts, payload
FROM events_kafka;
```

ClickHouse pulls messages in batches and writes them as parts efficiently.

## Tuning for High Throughput

Increase the number of background threads for merges and inserts:

```xml
<!-- config.xml -->
<background_pool_size>16</background_pool_size>
<background_schedule_pool_size>16</background_schedule_pool_size>
```

Adjust merge pressure settings:

```sql
SET max_insert_block_size = 1048576;
SET min_insert_block_size_rows = 1048576;
SET min_insert_block_size_bytes = 268435456;
```

## Monitoring Part Count

Watch for part accumulation:

```sql
SELECT table, count() AS parts, sum(rows) AS total_rows
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY parts DESC;
```

Alert if `parts` exceeds 300 for any table - that's the threshold before insert performance degrades.

## Summary

For high-frequency streaming data, use async inserts for simple pipelines, Buffer tables for application-level batching, and the Kafka engine for event stream integration. Always monitor part counts and tune background merge threads to keep ingestion stable under load.
