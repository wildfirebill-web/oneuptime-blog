# How to Use system.distribution_queue in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.distribution_queue, Distributed Table, Monitoring, Queue

Description: Learn how to use the system.distribution_queue table in ClickHouse to monitor pending data transfers from Distributed tables to their underlying shards.

---

When you insert data into a Distributed table in ClickHouse, the data is first buffered locally and then asynchronously sent to the appropriate shards. The `system.distribution_queue` table lets you monitor this pending delivery queue, diagnose delays, and detect failed transfers.

## What Is system.distribution_queue?

The `system.distribution_queue` table tracks data files that are waiting to be forwarded from a Distributed table to its underlying shard nodes. Each row represents a batch of data pending delivery to a specific shard.

## Basic Query

```sql
SELECT
    database,
    table,
    data_path,
    is_currently_sending,
    num_tries,
    last_exception,
    last_attempt_time,
    next_attempt_time
FROM system.distribution_queue
ORDER BY num_tries DESC;
```

## Monitor Queue Depth

To check how many files are waiting to be sent to each shard:

```sql
SELECT
    database,
    table,
    count() AS pending_files,
    sum(rows) AS pending_rows,
    sum(bytes) AS pending_bytes,
    formatReadableSize(sum(bytes)) AS pending_size
FROM system.distribution_queue
GROUP BY database, table
ORDER BY pending_files DESC;
```

## Identify Failed Deliveries

Persistent failures often indicate network issues or shard unavailability:

```sql
SELECT
    database,
    table,
    data_path,
    num_tries,
    last_exception,
    last_attempt_time
FROM system.distribution_queue
WHERE num_tries > 3
  AND last_exception != ''
ORDER BY num_tries DESC;
```

## Check Currently Sending Files

To see which files are actively being transferred right now:

```sql
SELECT
    database,
    table,
    data_path,
    num_tries
FROM system.distribution_queue
WHERE is_currently_sending = 1;
```

## Monitor Queue Growth Over Time

If the queue is growing, it could indicate that a shard is down or overloaded. Monitor the trend:

```sql
SELECT
    database,
    table,
    count() AS queue_depth
FROM system.distribution_queue
GROUP BY database, table;
```

Run this periodically and alert if `queue_depth` keeps increasing.

## Forcing Queue Flush

To manually trigger sending queued data (useful during maintenance or recovery):

```sql
SYSTEM FLUSH DISTRIBUTED distributed_events;
```

After flushing, verify the queue is clear:

```sql
SELECT count()
FROM system.distribution_queue
WHERE table = 'distributed_events';
```

## Understanding Retry Logic

ClickHouse retries failed shard deliveries with exponential backoff. The relevant columns are:

| Column | Description |
|---|---|
| `num_tries` | Number of delivery attempts so far |
| `last_exception` | Last error message |
| `last_attempt_time` | Timestamp of last delivery attempt |
| `next_attempt_time` | When the next retry is scheduled |

To cancel a stuck entry, you may need to manually remove the files from the data path on disk after identifying the shard target.

## Distributed Table Asynchronous Behavior

The `distributed_directory_monitor_sleep_time_ms` server setting controls how often ClickHouse checks the queue. The `distributed_directory_monitor_batch_inserts` setting enables batching multiple files into a single delivery for efficiency.

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'distributed_directory_monitor_sleep_time_ms',
    'distributed_directory_monitor_batch_inserts'
);
```

## Summary

The `system.distribution_queue` table is an essential diagnostic tool for ClickHouse clusters using Distributed tables. Use it to monitor delivery backlogs, identify failing shards, and ensure data reaches its destination reliably. Combine it with `SYSTEM FLUSH DISTRIBUTED` for operational control during incidents or maintenance windows.
