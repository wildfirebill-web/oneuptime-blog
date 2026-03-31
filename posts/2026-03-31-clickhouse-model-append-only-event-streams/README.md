# How to Model Append-Only Event Streams in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Append-Only, Event Stream, MergeTree, Immutable, Analytics

Description: Learn how to design append-only event stream tables in ClickHouse, leveraging immutability for maximum ingestion throughput and query performance.

---

Append-only event streams are ClickHouse's sweet spot. Logs, user events, metrics, traces, and audit records are naturally immutable - once written, they never change. Modeling these correctly unlocks maximum ingestion throughput and query performance.

## Why Append-Only Is Ideal for ClickHouse

```text
- No UPDATE or DELETE overhead
- Parts are immutable - no write amplification
- Perfect for columnar compression
- No concurrency conflicts
- Partitions can be dropped atomically for TTL
```

## Core Append-Only Table Design

```sql
CREATE TABLE user_events (
    event_id String,
    timestamp DateTime64(3),
    user_id UInt64,
    session_id String,
    event_type LowCardinality(String),
    page LowCardinality(String),
    referrer String,
    properties Map(String, String),
    -- Denormalized device context
    device_type LowCardinality(String),
    os LowCardinality(String),
    browser LowCardinality(String),
    country LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (user_id, timestamp, event_type)
TTL toDateTime(timestamp) + INTERVAL 1 YEAR;
```

## Ingestion Patterns for High Throughput

Batch inserts instead of row-by-row:

```bash
# Bulk insert via HTTP
curl -X POST 'http://localhost:8123/?query=INSERT+INTO+user_events+FORMAT+JSONEachRow' \
    --data-binary @events_batch.ndjson
```

Use async inserts to buffer small writes:

```sql
SET async_insert = 1;
SET async_insert_max_data_size = 10485760;
SET async_insert_busy_timeout_ms = 200;
```

## Deriving State from Events

Since events are immutable, derive current state using aggregations:

```sql
-- Current user state: last known values
SELECT
    user_id,
    argMax(country, timestamp) AS current_country,
    argMax(device_type, timestamp) AS last_device,
    max(timestamp) AS last_seen,
    count() AS total_events,
    countIf(event_type = 'purchase') AS purchase_count
FROM user_events
GROUP BY user_id;
```

## Session Analysis

Reconstruct sessions from append-only events:

```sql
SELECT
    user_id,
    session_id,
    min(timestamp) AS session_start,
    max(timestamp) AS session_end,
    dateDiff('second', min(timestamp), max(timestamp)) AS duration_sec,
    count() AS event_count,
    groupUniqArray(event_type) AS event_types
FROM user_events
WHERE timestamp >= today() - 7
GROUP BY user_id, session_id
ORDER BY session_start;
```

## Event Sourcing Pattern

Use events as the source of truth for entity state:

```sql
-- Orders as events
CREATE TABLE order_events (
    event_id String,
    order_id String,
    event_type LowCardinality(String),  -- created, paid, shipped, delivered, cancelled
    timestamp DateTime,
    actor_id UInt64,
    payload String
) ENGINE = MergeTree()
ORDER BY (order_id, timestamp);

-- Reconstruct current order state
SELECT
    order_id,
    argMax(event_type, timestamp) AS current_status,
    min(timestamp) AS created_at,
    max(timestamp) AS last_updated
FROM order_events
GROUP BY order_id;
```

## Partitioning Strategy for Append-Only Data

```text
Daily partitions (toYYYYMMDD):  best for logs, high ingestion rate
Monthly partitions (toYYYYMM):  better for long-term analytics, lower part count
Weekly partitions:              good middle ground
```

## Summary

Append-only event stream tables in ClickHouse are the simplest and highest-performing schema pattern. Never update or delete records - instead derive current state through aggregations using `argMax`, reconstruct sessions via GROUP BY, and use TTL for data lifecycle management. Batch ingestion with async inserts maximizes throughput.
