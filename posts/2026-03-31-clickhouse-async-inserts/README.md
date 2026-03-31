# How to Use Async Inserts in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, INSERT, Async Insert, Performance, Throughput

Description: Learn how async inserts buffer small writes in ClickHouse memory and flush them in batches, improving throughput for high-frequency insert workloads.

---

One of the most common ClickHouse performance pitfalls is high-frequency small inserts. Each INSERT normally creates a new part on disk, and hundreds of tiny inserts per second rapidly accumulate parts faster than the background merge process can consolidate them. Async inserts solve this by collecting small writes in a server-side buffer and flushing them as a single large insert. This post explains how async inserts work, how to configure them, and how to use them safely.

## How Async Inserts Work

When `async_insert = 1`, ClickHouse does not write data to disk immediately. Instead, it:

1. Accepts the INSERT and places the data in an in-memory buffer.
2. Returns an acknowledgment to the client (optionally waiting for flush if `wait_for_async_insert = 1`).
3. Flushes the buffer to disk as a single part when a threshold is met - either a time limit or a size limit.

This means many small inserts from many clients are coalesced into one large write, reducing part count and merge pressure.

## Enabling Async Inserts

Async inserts can be enabled per session or per query with a `SETTINGS` clause.

### Per query

```sql
INSERT INTO events (event_id, event_name, created_at)
VALUES (1, 'page_view', '2026-03-31 10:00:00')
SETTINGS async_insert = 1, wait_for_async_insert = 1;
```

### Per user (in users.xml or via SQL)

```sql
ALTER USER my_app_user SETTINGS async_insert = 1, wait_for_async_insert = 1;
```

### Server-wide default (in users.xml)

```xml
<profiles>
  <default>
    <async_insert>1</async_insert>
    <wait_for_async_insert>1</wait_for_async_insert>
  </default>
</profiles>
```

## Key Settings

### async_insert

Enables asynchronous insert mode.

```sql
SET async_insert = 1;
```

### wait_for_async_insert

- `1` (default): The client waits until the buffer is flushed to disk before receiving a success response. This provides write durability guarantees.
- `0`: Fire-and-forget. The client gets an immediate acknowledgment. Data may be lost if the server crashes before the flush.

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0; -- fire-and-forget mode
```

### async_insert_max_data_size

Maximum buffer size in bytes before a flush is triggered (default: 10 MB).

```sql
SET async_insert_max_data_size = 10485760; -- 10 MB
```

### async_insert_busy_timeout_ms

Maximum time in milliseconds the server waits before flushing a non-full buffer (default: 200 ms).

```sql
SET async_insert_busy_timeout_ms = 200;
```

## Practical Example

Imagine an application sending one event per HTTP request. Without async inserts, each request creates a part:

```bash
# Without async inserts - creates a part for every request
for i in $(seq 1 1000); do
    curl -s -X POST 'http://localhost:8123/' \
         --data-binary "INSERT INTO events FORMAT JSONEachRow
{\"event_id\":$i,\"event_name\":\"click\",\"created_at\":\"2026-03-31 10:00:00\"}"
done
```

With async inserts, all 1000 requests are buffered and flushed together:

```bash
# With async inserts - coalesced into far fewer parts
for i in $(seq 1 1000); do
    curl -s -X POST 'http://localhost:8123/?async_insert=1&wait_for_async_insert=1' \
         --data-binary "INSERT INTO events FORMAT JSONEachRow
{\"event_id\":$i,\"event_name\":\"click\",\"created_at\":\"2026-03-31 10:00:00\"}"
done
```

## Monitoring Async Insert Buffers

Check the current state of pending async insert buffers:

```sql
SELECT
    database,
    table,
    format,
    rows,
    bytes,
    timeout_milliseconds,
    create_time
FROM system.asynchronous_insert_log
ORDER BY create_time DESC
LIMIT 20;
```

Track flush history and any errors:

```sql
SELECT
    event_time,
    database,
    table,
    rows,
    exception
FROM system.asynchronous_insert_log
WHERE status != 'Ok'
ORDER BY event_time DESC
LIMIT 10;
```

## Deduplication with Async Inserts

By default, ClickHouse deduplicates synchronous inserts using an insert block checksum. With async inserts, multiple client requests may be merged into one block, so the deduplication behavior differs.

To enable deduplication for async inserts:

```sql
SET async_insert_deduplicate = 1;
```

Each individual client insert is assigned a deduplication token. If the same data is re-sent (e.g., on a retry after a timeout), ClickHouse will not insert it twice.

```sql
INSERT INTO events VALUES (1, 'page_view', '2026-03-31 10:00:00')
SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,
    async_insert_deduplicate = 1;
```

## When to Use Async Inserts

- Many clients each sending small payloads (IoT sensors, application event tracking, log shippers).
- When you cannot batch on the client side.
- As an alternative to a Buffer table when you prefer server-managed buffering.

Avoid async inserts when you need strict transactional guarantees or when you are already batching large payloads on the client.

## Summary

Async inserts let ClickHouse absorb high-frequency small writes by buffering them in memory and flushing them as a single large part. Use `async_insert = 1` with `wait_for_async_insert = 1` for durable writes, tune `async_insert_max_data_size` and `async_insert_busy_timeout_ms` to control flush thresholds, and enable `async_insert_deduplicate` when retries are possible. This feature is the recommended solution for high-cardinality, low-volume-per-request write workloads.
