# How to Use system.events for Cumulative Counters in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.events, Cumulative Counter, Observability, Monitoring, Performance

Description: Query system.events to access cumulative event counters like query counts, cache hits, network bytes, and merge counts since the ClickHouse server last started.

---

ClickHouse maintains hundreds of cumulative event counters internally that track everything from queries executed to bytes written to cache hit rates. These are exposed through `system.events`, making it a powerful source for computing rates and understanding server-wide activity trends.

## What is system.events?

`system.events` is a read-only system table with one row per event type. Values are monotonically increasing counters that reset only on server restart. Key columns:

- `event` - event name (e.g., `Query`, `SelectQuery`, `InsertQuery`)
- `value` - cumulative count since last restart
- `description` - human-readable explanation

## Basic Query

```sql
SELECT event, value, description
FROM system.events
ORDER BY value DESC
LIMIT 30;
```

## Key Cumulative Counters

```sql
SELECT event, value
FROM system.events
WHERE event IN (
    'Query',
    'SelectQuery',
    'InsertQuery',
    'FailedQuery',
    'QueryTimeMicroseconds',
    'DiskReadElapsedMicroseconds',
    'DiskWriteElapsedMicroseconds',
    'NetworkReceiveBytes',
    'NetworkSendBytes',
    'MergedRows',
    'MarkCacheHits',
    'MarkCacheMisses'
)
ORDER BY event;
```

## Computing Event Rates

Since values are cumulative, compute rates by sampling at two points:

```sql
-- First sample (snapshot)
CREATE TABLE IF NOT EXISTS events_snapshot (
    ts DateTime,
    event String,
    value UInt64
) ENGINE = Memory;

INSERT INTO events_snapshot SELECT now(), event, value FROM system.events;

-- Wait 60 seconds, then compute rate:
SELECT
    e.event,
    (e.value - s.value) / 60 AS per_second_rate
FROM system.events e
JOIN events_snapshot s ON e.event = s.event
WHERE e.event IN ('Query', 'SelectQuery', 'InsertQuery', 'FailedQuery')
ORDER BY per_second_rate DESC;
```

## Cache Hit Rate

Calculate mark cache effectiveness:

```sql
SELECT
    sumIf(value, event = 'MarkCacheHits')                               AS hits,
    sumIf(value, event = 'MarkCacheMisses')                             AS misses,
    round(hits / (hits + misses) * 100, 2)                              AS hit_rate_pct
FROM system.events
WHERE event IN ('MarkCacheHits', 'MarkCacheMisses');
```

## Query Failure Rate

```sql
SELECT
    sumIf(value, event = 'FailedQuery')  AS failed,
    sumIf(value, event = 'Query')        AS total,
    round(failed / total * 100, 3)       AS failure_pct
FROM system.events
WHERE event IN ('FailedQuery', 'Query');
```

## Difference from system.metrics

`system.events` stores cumulative totals (like an odometer) - they only go up. `system.metrics` stores current instantaneous values (like a speedometer). Use `system.events` to compute throughput rates and totals, and `system.metrics` for live current-state gauges.

## Integration with Prometheus

The ClickHouse Prometheus exporter exposes `system.events` counters as Prometheus counters, enabling Grafana dashboards with rate() functions that derive per-second rates automatically.

## Summary

`system.events` provides cumulative server-wide activity counters in ClickHouse. Use it to compute query rates, cache hit ratios, failure percentages, and I/O throughput by sampling values at intervals. It complements `system.metrics` (live gauges) and `system.asynchronous_metrics` (background refresh) for complete server observability.
