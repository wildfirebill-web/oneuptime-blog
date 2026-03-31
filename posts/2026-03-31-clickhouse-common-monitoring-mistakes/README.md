# Common ClickHouse Monitoring Mistakes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Monitoring, Prometheus, Observability, System Table

Description: Avoid the most common ClickHouse monitoring mistakes that leave clusters blind to performance degradation, replication lag, and disk pressure.

---

ClickHouse exposes rich internal metrics but most deployments only scratch the surface. These monitoring mistakes leave operators unaware of problems until users complain.

## Mistake 1: Only Monitoring Server Uptime

Checking that the HTTP endpoint returns 200 tells you ClickHouse is running - it does not tell you whether queries are completing, merges are keeping up, or replication is healthy.

```sql
-- Add these to your monitoring stack
SELECT metric, value FROM system.metrics
WHERE metric IN (
  'BackgroundMergesAndMutationsPoolTask',
  'ReplicatedChecks',
  'Query',
  'DelayedInserts'
);
```

Alert on `DelayedInserts > 0` as an early warning sign of merge backlog.

## Mistake 2: Not Monitoring Part Count Per Table

ClickHouse merges parts in the background. When merges fall behind inserts, part count grows until queries slow down and new inserts are throttled.

```sql
SELECT database, table, count() AS parts, sum(rows) AS total_rows
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY parts DESC
LIMIT 20;
```

Alert when any table exceeds 300 active parts.

## Mistake 3: Ignoring the Merge Queue Length

A growing merge queue means merges cannot keep up with inserts. This is the precursor to "too many parts" errors.

```sql
SELECT count() AS pending_merges
FROM system.merges;
```

Track this metric over time and alert when it consistently exceeds 100.

## Mistake 4: Not Tracking Query Duration Percentiles

Average query duration hides tail latency problems. Track p95 and p99 from `system.query_log`.

```sql
SELECT
  quantile(0.95)(query_duration_ms) AS p95_ms,
  quantile(0.99)(query_duration_ms) AS p99_ms,
  count() AS query_count
FROM system.query_log
WHERE event_date = today() AND type = 'QueryFinish';
```

## Mistake 5: Not Monitoring Disk Usage Per Volume

ClickHouse uses tiered storage. Monitoring only total disk usage misses the case where the hot tier fills up while cold storage is empty.

```sql
SELECT name, path, free_space, total_space,
  round((1 - free_space / total_space) * 100, 1) AS used_pct
FROM system.disks;
```

Alert when any disk exceeds 85% usage.

## Mistake 6: Missing Replication Queue Errors

Replication errors accumulate silently. A table can be months behind on one replica while queries return stale data.

```sql
SELECT database, table, count() AS error_count, max(last_exception) AS last_error
FROM system.replication_queue
WHERE last_exception != ''
GROUP BY database, table;
```

Alert on any non-empty `last_exception` in the replication queue.

## Summary

Effective ClickHouse monitoring requires tracking part count, merge queue depth, query duration percentiles, disk usage per volume, and replication queue errors - not just server uptime. Export these from `system.*` tables into Prometheus or your preferred metrics backend and configure alerts before issues become outages.
