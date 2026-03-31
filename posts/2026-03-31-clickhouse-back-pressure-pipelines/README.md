# How to Handle Back-Pressure in ClickHouse Data Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Back-Pressure, Data Pipeline, Async Insert, Reliability

Description: Learn how to handle back-pressure in ClickHouse ingestion pipelines using async inserts, queues, retry logic, and flow control techniques.

---

Back-pressure occurs when ClickHouse cannot absorb data as fast as it arrives. Without proper handling, you risk data loss, memory exhaustion, or cascading failures upstream.

## Root Causes of Back-Pressure

- Too many small inserts creating excessive parts
- ClickHouse merge saturation (too many parts per partition)
- Network or disk I/O bottleneck
- Insufficient ClickHouse compute or memory

## Using Async Inserts to Buffer at the Database Level

Async inserts queue data server-side and flush when thresholds are met:

```sql
SET async_insert = 1;
SET async_insert_max_data_size = '10M';
SET async_insert_busy_timeout_ms = 500;
SET async_insert_max_query_number = 1000;
```

With `wait_for_async_insert = 0`, clients receive an immediate 200 OK without blocking on the flush.

## Queue-Based Pipeline

Place a durable queue (e.g., Kafka, Redis Streams) between the source and ClickHouse:

```text
App --> Kafka Topic --> Consumer --> Batch Buffer --> ClickHouse
```

The consumer pulls batches and applies back-pressure via Kafka consumer lag:

```bash
# Monitor lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group clickhouse-consumer
```

If lag grows, scale the consumer horizontally.

## Retry with Exponential Backoff

When ClickHouse returns a 503 or `TOO_MANY_PARTS` error, retry with backoff:

```python
import time

def insert_with_retry(client, rows, max_retries=5):
    delay = 1
    for attempt in range(max_retries):
        try:
            client.insert('events', rows)
            return
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(delay)
            delay = min(delay * 2, 60)
```

## Monitoring Parts Count

Watch for too many active parts as an early back-pressure signal:

```sql
SELECT database, table, count() AS active_parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
HAVING active_parts > 100
ORDER BY active_parts DESC;
```

## Throttling Inserts

Rate-limit your producer to match ClickHouse's sustainable throughput:

```python
import time

RATE_LIMIT = 500  # rows/sec

def throttled_insert(rows):
    start = time.time()
    client.insert('events', rows)
    elapsed = time.time() - start
    expected = len(rows) / RATE_LIMIT
    if elapsed < expected:
        time.sleep(expected - elapsed)
```

## Summary

Handle ClickHouse back-pressure by enabling async inserts, inserting through a durable queue, retrying with exponential backoff, and monitoring active parts count. Throttle producers to match sustainable write throughput and scale consumers horizontally when lag grows.
