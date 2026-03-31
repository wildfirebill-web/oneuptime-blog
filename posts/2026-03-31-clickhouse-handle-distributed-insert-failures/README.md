# How to Handle Distributed INSERT Failures in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed Table, Insert, Error Handling, Reliability

Description: Learn how ClickHouse handles failed inserts on Distributed tables, how to monitor the distribution queue, and how to recover from shard unavailability.

---

When inserting into a ClickHouse Distributed table, the data is routed to one or more shards. If a shard is unavailable during an asynchronous insert, ClickHouse spools the data locally and retries. Understanding this failure mode helps you build reliable ingestion pipelines.

## How the Distributed Spool Works

When a shard is unreachable, ClickHouse writes insert data to a local spool directory:

```bash
ls /var/lib/clickhouse/data/default/distributed_events/
# shard1/  shard2/  shard3/
```

Each subdirectory corresponds to a shard. Failed sends accumulate here until the shard recovers.

## Monitoring the Distribution Queue

Check the current state of pending inserts:

```sql
SELECT
    database,
    table,
    host_name,
    host_port,
    data_path,
    task_count,
    failed_count,
    error
FROM system.distribution_queue
WHERE failed_count > 0
ORDER BY failed_count DESC;
```

A rising `failed_count` with non-empty `error` means a shard is consistently unreachable.

## Checking Error Logs

View recent distribution errors:

```sql
SELECT
    event_time,
    message
FROM system.text_log
WHERE message LIKE '%distribution%'
  AND level IN ('Error', 'Warning')
ORDER BY event_time DESC
LIMIT 20;
```

## Configuring Retry Behavior

Tune how aggressively ClickHouse retries failed shard sends in `config.xml`:

```xml
<distributed_directory_monitor_sleep_time_ms>500</distributed_directory_monitor_sleep_time_ms>
<distributed_directory_monitor_max_sleep_time_ms>30000</distributed_directory_monitor_max_sleep_time_ms>
```

The retry interval starts at `sleep_time_ms` and doubles up to `max_sleep_time_ms` with each failure.

## Setting Insert Error Handling Mode

Control what happens when a shard is unreachable during a synchronous insert:

```sql
-- Throw an error if any shard is unavailable
SET insert_distributed_one_random_shard = 0;

-- Skip unavailable shards and insert to available ones only
SET distributed_push_down_limit = 1;
```

For async inserts, configure the `fsync_directories` option to ensure spool data survives a node restart:

```xml
<fsync_directories>1</fsync_directories>
```

## Manually Flushing the Spool

Once a shard recovers, trigger an immediate flush instead of waiting for the background retry:

```sql
SYSTEM FLUSH DISTRIBUTED distributed_events;
```

This forces ClickHouse to attempt sending all spooled data immediately.

## Dropping Stuck Spool Files

If spool files are corrupted or you decide to discard them:

```bash
# On the inserting node
sudo systemctl stop clickhouse-server
sudo rm /var/lib/clickhouse/data/default/distributed_events/shard1/*.bin
sudo systemctl start clickhouse-server
```

Only do this if you are certain the data can be re-inserted from a source system.

## Preventing Spool Overflow

Set a maximum spool size to prevent disk exhaustion:

```xml
<max_distributed_connections>1024</max_distributed_connections>
```

Alert when spool files grow too large:

```sql
SELECT
    database,
    table,
    host_name,
    task_count
FROM system.distribution_queue
WHERE task_count > 10000;
```

## Summary

ClickHouse handles Distributed INSERT failures by spooling data locally and retrying asynchronously. Monitor `system.distribution_queue` for failed sends, use `SYSTEM FLUSH DISTRIBUTED` to trigger immediate retries, and configure retry intervals to match your shard recovery SLA. Enable `fsync_directories` to protect spooled data across node restarts.
