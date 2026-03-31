# How to Use system.metrics for Real-Time Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.metrics, Real-Time Monitoring, Observability, Performance, System Table

Description: Use system.metrics to query real-time point-in-time server metrics like active connections, running queries, and memory usage in ClickHouse.

---

ClickHouse exposes hundreds of internal performance counters through system tables. `system.metrics` provides a snapshot of current real-time gauges - values that reflect the instantaneous state of the server at the moment you query it.

## What is system.metrics?

`system.metrics` contains point-in-time metric values. Unlike `system.events` (cumulative counters) or `system.asynchronous_metrics` (background-refreshed metrics), the values in `system.metrics` represent the current live state.

Each row has:
- `metric` - metric name (e.g., `Query`, `Connection`, `MemoryTracking`)
- `value` - current numeric value
- `description` - human-readable explanation

## Basic Query

```sql
SELECT metric, value, description
FROM system.metrics
ORDER BY metric;
```

## Key Real-Time Metrics

Query the most operationally important metrics:

```sql
SELECT metric, value
FROM system.metrics
WHERE metric IN (
    'Query',
    'BackgroundMergesAndMutationsPoolTask',
    'BackgroundFetchesPoolTask',
    'DistributedSend',
    'MemoryTracking',
    'OpenFileForRead',
    'OpenFileForWrite',
    'ReplicatedChecks',
    'Connection'
)
ORDER BY metric;
```

## Monitoring Active Queries

```sql
SELECT value AS active_queries
FROM system.metrics
WHERE metric = 'Query';
```

Compare this against your expected concurrent query load. Spikes in `Query` count may indicate slow queries piling up.

## Memory Usage

```sql
SELECT
    metric,
    formatReadableSize(value) AS current_value
FROM system.metrics
WHERE metric LIKE '%Memory%'
ORDER BY value DESC;
```

## Background Pool Utilization

```sql
SELECT
    metric,
    value
FROM system.metrics
WHERE metric LIKE '%Pool%'
ORDER BY metric;
```

High values in `BackgroundMergesAndMutationsPoolTask` relative to the configured pool size (`background_pool_size`) indicate merge contention.

## Building a Real-Time Dashboard

Combine `system.metrics` with a time series by polling it from an external tool (Prometheus, Grafana):

```sql
-- Poll this query every 15 seconds
SELECT
    now()   AS ts,
    metric,
    value
FROM system.metrics
WHERE metric IN ('Query', 'MemoryTracking', 'BackgroundMergesAndMutationsPoolTask');
```

The ClickHouse Prometheus exporter scrapes `system.metrics` automatically if configured.

## Difference from system.events and system.asynchronous_metrics

| Table | Type | Reset |
|-------|------|-------|
| `system.metrics` | Current value (gauge) | Never - always live |
| `system.events` | Cumulative counter | On server restart |
| `system.asynchronous_metrics` | Background refresh | Periodically |

Use `system.metrics` for live operational dashboards, `system.events` for rate calculations, and `system.asynchronous_metrics` for slower-changing server health indicators.

## Summary

`system.metrics` is ClickHouse's real-time gauge table, ideal for monitoring current active queries, memory pressure, connection counts, and background task pool utilization. Poll it regularly from your monitoring stack to build live operational dashboards without any overhead to query processing.
