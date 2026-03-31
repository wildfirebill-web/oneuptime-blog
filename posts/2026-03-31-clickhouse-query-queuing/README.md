# How to Implement Query Queuing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Query Queuing, Concurrency, Resource Management, Settings Profile, Performance

Description: Learn how to implement query queuing in ClickHouse to prevent resource exhaustion when concurrent query load exceeds server capacity.

---

When more queries arrive than ClickHouse can execute simultaneously, you need queuing to prevent OOM errors and degraded performance. ClickHouse provides query queue settings that hold excess queries in a waiting state rather than rejecting them outright.

## The Problem Without Queuing

Without limits, too many concurrent queries can exhaust memory and CPU, causing:
- `Memory limit exceeded` errors
- Server slowdowns affecting all users
- Cascading failures in dashboards and applications

## Key Queue Settings

The primary settings for query queuing are:

| Setting | Description |
|---------|-------------|
| `max_concurrent_queries` | Total concurrent queries the server accepts |
| `max_concurrent_queries_for_user` | Per-user concurrent query limit |
| `max_waiting_queries` | Max queries allowed to wait in the queue |
| `min_execution_speed_bytes` | Cancel queries that run too slowly |

## Configuring Global Queue Limits

In `config.xml` or via `clickhouse-client` with admin rights:

```xml
<clickhouse>
    <max_concurrent_queries>100</max_concurrent_queries>
    <max_concurrent_queries_for_user>20</max_concurrent_queries_for_user>
    <max_waiting_queries>50</max_waiting_queries>
</clickhouse>
```

Or set dynamically:

```sql
SET max_concurrent_queries = 100;
```

## Per-User Queue Limits via Settings Profiles

```sql
CREATE SETTINGS PROFILE dashboard_user_profile
SETTINGS
    max_concurrent_queries_for_user = 10,
    max_execution_time = 30;

ALTER USER dashboard_user SETTINGS PROFILE 'dashboard_user_profile';
```

## Monitoring the Query Queue

Check how many queries are currently running or waiting:

```sql
SELECT
    status,
    count() AS cnt,
    max(elapsed) AS max_elapsed_sec
FROM system.processes
GROUP BY status;
```

View currently queued queries:

```sql
SELECT
    query_id,
    user,
    query,
    elapsed,
    memory_usage
FROM system.processes
WHERE status = 'waiting'
ORDER BY elapsed DESC;
```

## Configuring Queue Timeout

Queries that wait too long should be rejected rather than staying in queue indefinitely:

```sql
-- Reject a query if it waits more than 10 seconds in the queue
SET queue_max_wait_ms = 10000;
```

This prevents queue buildup during prolonged overload.

## Workload Scheduling Integration

For more advanced queuing with priorities, integrate with workload scheduling:

```sql
CREATE WORKLOAD interactive PARENT default SETTINGS
    priority = 0,
    max_concurrent_queries = 20;

CREATE WORKLOAD batch PARENT default SETTINGS
    priority = 10,
    max_concurrent_queries = 5;
```

Assign queries to workloads by setting `workload` at query time:

```sql
SET workload = 'interactive';
SELECT count() FROM events WHERE timestamp >= today();
```

## Detecting Queue Saturation

Alert when the queue is frequently full:

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    countIf(type = 'QueryFinish') AS finished,
    countIf(exception LIKE '%Too many simultaneous%') AS rejected
FROM system.query_log
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

## Summary

ClickHouse query queuing is controlled via `max_concurrent_queries`, `max_waiting_queries`, and `queue_max_wait_ms`. Apply per-user limits through settings profiles and use `system.processes` to monitor the queue in real time. For priority-based queuing, combine with the workload scheduling feature.
