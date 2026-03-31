# How to Optimize Distributed INSERT INTO in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed Table, INSERT, Write Performance, Sharding

Description: Learn how to optimize distributed INSERT INTO operations in ClickHouse to reduce write amplification and maximize ingestion throughput.

---

## How Distributed Inserts Work

When you INSERT into a ClickHouse Distributed table, data is first written to a local shard buffer, then asynchronously forwarded to the correct remote shards based on the sharding key. Understanding this two-stage process is key to tuning write performance.

## Synchronous vs Asynchronous Insert Modes

By default, Distributed table inserts are asynchronous:

```sql
-- Asynchronous (default): INSERT returns immediately, forwarding happens in background
INSERT INTO distributed_http_logs VALUES (...);
```

For stronger durability guarantees, use synchronous mode:

```sql
INSERT INTO distributed_http_logs
SETTINGS distributed_foreground_insert = 1
VALUES (...);
```

## Insert Directly to Local Tables for Maximum Performance

The fastest pattern is to bypass the Distributed table and insert directly to the local shard tables:

```sql
-- Insert directly to the local table on each shard
INSERT INTO local_http_logs VALUES (...);
```

Use a load balancer or application-level shard routing to distribute inserts across shards. This eliminates the forwarding overhead entirely.

## Batching for High Throughput

Small, frequent inserts create many small parts and degrade performance. Batch inserts to at least 1,000-100,000 rows per INSERT:

```sql
INSERT INTO distributed_http_logs
SELECT * FROM incoming_batch_staging
WHERE batch_id = 12345;
```

Configure async_insert for client-side batching:

```sql
INSERT INTO distributed_http_logs
SETTINGS async_insert = 1, wait_for_async_insert = 0
VALUES (...);
```

## Tuning Distributed Insert Buffer

Control how long data waits before being forwarded. In the server config:

```xml
<distributed_directory_monitor_sleep_time_ms>500</distributed_directory_monitor_sleep_time_ms>
<distributed_directory_monitor_max_sleep_time_ms>5000</distributed_directory_monitor_max_sleep_time_ms>
```

Reduce sleep time for lower latency forwarding; increase it to reduce overhead under high write load.

## Monitoring Distributed Insert Queue

Check the backlog of pending forwards:

```sql
SELECT
    database,
    table,
    active_parts,
    total_rows,
    formatReadableSize(total_bytes) AS pending_bytes
FROM system.distribution_queue
ORDER BY total_bytes DESC;
```

A growing queue indicates forwarding is slower than ingestion.

## Handling Insert Failures

Check for stuck forwards:

```sql
SELECT *
FROM system.distribution_queue
WHERE broken_data_files > 0;
```

Broken files indicate forwarding errors requiring manual intervention.

## Summary

Optimize distributed inserts by batching data into large chunks, considering direct-to-local-table inserts for maximum throughput, and tuning the distribution monitor intervals for your latency vs. throughput trade-off. Monitor system.distribution_queue to catch forwarding backlogs before they impact ingestion pipelines.
