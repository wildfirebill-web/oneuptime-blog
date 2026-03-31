# How to Use Async Inserts in ClickHouse for High Ingestion

Author: [oneuptime](https://www.github.com/oneuptime)

Tags: ClickHouse, Performance, Database, Analytics, Infrastructure

Description: Learn how to use ClickHouse async inserts to absorb high-frequency small inserts, reduce part creation, and sustain high ingestion rates without a separate buffer layer.

## Introduction

ClickHouse async inserts allow you to send small, frequent inserts to the database without creating a new data part for each one. Instead, ClickHouse buffers the data in memory across multiple insert statements and flushes it to disk as a single larger part once a time or size threshold is reached. This feature is ideal when you cannot batch on the client side - for example, when events arrive one at a time from a large fleet of agents.

Async inserts were introduced in ClickHouse 21.11 and are stable as of 22.8.

## How Async Inserts Work

Without async inserts:
1. Client sends INSERT with 100 rows.
2. ClickHouse writes a new data part immediately.
3. 1,000 clients sending every second = 1,000 parts per second. Merge queue overwhelmed.

With async inserts:
1. Client sends INSERT with 100 rows.
2. ClickHouse holds the data in a server-side buffer.
3. When the buffer reaches `async_insert_max_data_size` or `async_insert_busy_timeout_ms` expires, ClickHouse flushes all buffered data from all clients as one large part.
4. 1,000 clients sending every second = 1 flush per second = 1 part per second. Merge queue healthy.

## Enabling Async Inserts

Async inserts are enabled at the session or user profile level.

```sql
-- Enable for the current session
SET async_insert = 1;
SET wait_for_async_insert = 0;  -- fire-and-forget (higher throughput)
-- or
SET wait_for_async_insert = 1;  -- wait for flush confirmation (lower throughput, higher durability)

-- Then insert as normal
INSERT INTO events (service, event_time, value)
VALUES ('api', now(), 42.5);
```

Enable globally for a user profile in `users.xml`:

```xml
<profiles>
    <ingestion_agent>
        <async_insert>1</async_insert>
        <wait_for_async_insert>0</wait_for_async_insert>
        <async_insert_max_data_size>10485760</async_insert_max_data_size>
        <async_insert_busy_timeout_ms>200</async_insert_busy_timeout_ms>
    </ingestion_agent>
</profiles>
```

## Key Async Insert Settings

```sql
-- Maximum buffer size per table before forced flush (default: 10 MB)
SET async_insert_max_data_size = 10485760;  -- 10 MB

-- Maximum time to wait before flushing even if buffer is not full (default: 200 ms)
SET async_insert_busy_timeout_ms = 200;

-- Maximum number of queries waiting in the async queue (default: 10)
SET async_insert_max_query_number = 100;

-- Whether to wait for the data to be flushed to disk before returning
SET wait_for_async_insert = 0;  -- async (returns immediately)
SET wait_for_async_insert = 1;  -- sync (returns after flush, confirms durability)

-- Timeout for wait_for_async_insert = 1 (default: 120 seconds)
SET wait_for_async_insert_timeout = 10;
```

## Practical Configuration for Different Ingestion Patterns

### High-Frequency Small Events (Application Logging)

When thousands of application instances each send 1-10 rows per second:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;
SET async_insert_busy_timeout_ms = 500;
SET async_insert_max_data_size = 5242880;  -- 5 MB
```

### Moderate Volume with Durability Requirement

When you need confirmation that data was persisted (e.g., payment events):

```sql
SET async_insert = 1;
SET wait_for_async_insert = 1;
SET wait_for_async_insert_timeout = 5;
SET async_insert_busy_timeout_ms = 100;
SET async_insert_max_data_size = 10485760;
```

### Batch Pipeline Sending Large Inserts

For pipelines that already batch (e.g., 100K rows at a time), async inserts add little value and can slightly increase latency:

```sql
SET async_insert = 0;
```

## Monitoring Async Insert Activity

```sql
-- Check async insert queue and flush statistics
SELECT
    table,
    format,
    status,
    elapsed_microseconds,
    bytes,
    rows,
    exception
FROM system.asynchronous_insert_log
WHERE event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC
LIMIT 50;

-- Aggregate: flushes per minute and bytes flushed
SELECT
    toStartOfMinute(event_time) AS minute,
    count()                     AS flush_count,
    sum(rows)                   AS rows_inserted,
    formatReadableSize(sum(bytes)) AS bytes_flushed
FROM system.asynchronous_insert_log
WHERE status = 'Ok'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;

-- Failed async inserts
SELECT
    event_time,
    table,
    bytes,
    rows,
    exception
FROM system.asynchronous_insert_log
WHERE status != 'Ok'
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY event_time DESC;
```

## Deduplication with Async Inserts

By default, async inserts do not perform insert deduplication. If your pipeline may retry inserts, enable deduplication:

```sql
SET async_insert_deduplicate = 1;
```

With deduplication enabled, ClickHouse tracks insert checksums and discards duplicate inserts within the deduplication window.

## Example: High-Throughput Agent Ingestion

```python
import clickhouse_driver
import time
import random

client = clickhouse_driver.Client(
    host='clickhouse.internal',
    settings={
        'async_insert': 1,
        'wait_for_async_insert': 0,
        'async_insert_busy_timeout_ms': 200,
        'async_insert_max_data_size': 10 * 1024 * 1024,
    }
)

services = ['api', 'web', 'worker', 'scheduler']

for i in range(10000):
    client.execute(
        'INSERT INTO events (service, value, event_time) VALUES',
        [{'service': random.choice(services),
          'value': random.random() * 100,
          'event_time': int(time.time())}]
    )

print("All inserts submitted to async buffer")
```

## Comparing Sync vs Async Insert Performance

```bash
# Without async: 10,000 single-row inserts (creates 10,000 parts)
time for i in $(seq 1 10000); do
    clickhouse-client --query "INSERT INTO events VALUES ('api', now(), 1.0)"
done
# Result: ~45 seconds, 10,000 parts created

# With async inserts enabled
time for i in $(seq 1 10000); do
    clickhouse-client --query "INSERT INTO events VALUES ('api', now(), 1.0)" \
        --async_insert=1 --wait_for_async_insert=0
done
# Result: ~3 seconds, ~15 parts created (one per 200ms flush window)
```

## Summary

Async inserts solve the fundamental tension between high-frequency small inserts and ClickHouse's part-based storage model. By buffering inserts server-side and flushing them as large parts, async inserts let you ingest from thousands of agents without creating thousands of parts per second. Configure `async_insert_busy_timeout_ms` to control latency and `async_insert_max_data_size` to control batch size, monitor with `system.asynchronous_insert_log`, and use `wait_for_async_insert = 1` when durability confirmation is required.
