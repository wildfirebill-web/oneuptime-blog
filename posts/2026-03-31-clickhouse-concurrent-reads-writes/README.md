# How to Handle Concurrent Reads and Writes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Concurrency, Read, Write, MergeTree, Locking, Performance

Description: Understand how ClickHouse handles concurrent reads and writes, and configure isolation, buffer tables, and async inserts to maximize throughput.

---

ClickHouse uses a multi-version concurrency model for MergeTree tables. Reads and writes do not block each other - a query always sees a consistent snapshot of the table at the moment it starts. Understanding this model helps you design ingestion pipelines that coexist with analytical queries without contention.

## How Concurrent Access Works

MergeTree stores data as immutable parts. When you INSERT, ClickHouse creates a new part. When you SELECT, ClickHouse reads the set of active parts at the start of the query. No row-level locks are needed because parts are never modified in place.

```sql
-- These run concurrently with no blocking
INSERT INTO events SELECT * FROM input_stream;
SELECT count() FROM events WHERE event_date = today();
```

## Part Count and Read Performance

Heavy concurrent INSERT workloads create many small parts, which slows reads because more parts must be merged at query time. Minimize parts by batching inserts:

```sql
-- Bad: too many single-row inserts
INSERT INTO events VALUES (now(), 1, 'click');
INSERT INTO events VALUES (now(), 2, 'view');

-- Good: one batch insert
INSERT INTO events VALUES
(now(), 1, 'click'),
(now(), 2, 'view'),
(now(), 3, 'purchase');
```

## Async Inserts to Buffer Small Writes

For high-frequency small inserts from many clients, use async inserts to let the server batch them:

```sql
SET async_insert = 1;
SET async_insert_max_data_size = 10485760; -- 10 MB
SET async_insert_busy_timeout_ms = 500;
SET wait_for_async_insert = 0;
```

The server accumulates rows and flushes them as a single part, keeping part count low.

## Buffer Tables for Write Spikes

A Buffer table absorbs bursts and periodically flushes to the underlying table:

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(default, events, 16, 10, 60, 1000, 100000, 1000000, 10000000);
```

Parameters: `num_layers, min_time, max_time, min_rows, max_rows, min_bytes, max_bytes`. Inserts go to the buffer; ClickHouse flushes when any threshold is met.

## Monitoring Concurrent Activity

```sql
SELECT
    query,
    elapsed,
    read_rows,
    memory_usage,
    query_kind
FROM system.processes
ORDER BY elapsed DESC;
```

## Parts Count Health Check

```sql
SELECT
    table,
    count()             AS active_parts,
    sum(rows)           AS total_rows,
    max(rows)           AS max_part_rows
FROM system.parts
WHERE database = 'default' AND active
GROUP BY table
ORDER BY active_parts DESC;
```

More than a few hundred active parts per table indicates inserts are creating parts faster than merges can consolidate them.

## Summary

ClickHouse concurrent reads and writes are naturally non-blocking due to the immutable-part model. The key operational concern is part count: too many small parts slow reads. Use batched inserts, async inserts, or Buffer tables to maintain a healthy part count, and monitor `system.parts` and `system.processes` to detect contention early.
