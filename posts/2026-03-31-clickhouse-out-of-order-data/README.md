# How to Handle Out-of-Order Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Out-of-Order, Data Ingestion, MergeTree, Streaming

Description: Learn how to design ClickHouse tables and ingestion pipelines to handle out-of-order event data without sacrificing query accuracy.

---

## Why Out-of-Order Data Happens

Distributed systems generate events out of order due to producer retries, clock skew, and batching delays. ClickHouse stores data in parts sorted by the primary key, so out-of-order timestamps in the primary key can degrade compression and query performance.

## Primary Key Design

Avoid putting a high-cardinality monotonically increasing column first in the sort key. Instead, lead with lower-cardinality dimensions:

```sql
CREATE TABLE metrics
(
    service     LowCardinality(String),
    metric_name LowCardinality(String),
    ts          DateTime64(3),
    value       Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (service, metric_name, ts);
```

With `service` and `metric_name` first, out-of-order `ts` values within a service-metric pair still compress and sort well within their range.

## Using a Buffer Table

A `Buffer` table absorbs short bursts of out-of-order data and flushes sorted batches to the main table:

```sql
CREATE TABLE metrics_buffer AS metrics
ENGINE = Buffer(default, metrics, 4, 5, 60, 10000, 1000000, 10000000, 100000000);
```

Writes go to `metrics_buffer` first. ClickHouse flushes automatically when time or size thresholds are met.

## Ingestion-Side Sorting

Sort data at the producer before writing to ClickHouse. In Python with the `clickhouse-driver`:

```python
import clickhouse_driver

rows.sort(key=lambda r: (r["service"], r["metric_name"], r["ts"]))

client = clickhouse_driver.Client("localhost")
client.execute(
    "INSERT INTO metrics (service, metric_name, ts, value) VALUES",
    rows
)
```

Pre-sorting reduces the work ClickHouse does during part merges.

## Monitoring Part Counts

Too many small parts from out-of-order inserts increase merge pressure. Monitor part counts:

```sql
SELECT table, count() AS parts, sum(rows) AS total_rows
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY parts DESC;
```

If parts grow fast, increase `max_insert_block_size` or use asynchronous inserts.

## Async Inserts for Micro-Batches

Async inserts buffer rows server-side before forming parts:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;

INSERT INTO metrics VALUES (...)
```

This lets ClickHouse accumulate and sort small out-of-order batches before writing them as a single part.

## Summary

Handling out-of-order data in ClickHouse comes down to smart primary key design, producer-side sorting, and using buffer layers or async inserts to smooth write patterns. Monitoring part counts helps you catch ingestion issues before they affect merge performance.
