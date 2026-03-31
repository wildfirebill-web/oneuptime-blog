# How to Monitor ClickHouse Query Latency Percentiles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Monitoring, Query Latency, Percentile, system.query_log, Performance

Description: Learn how to monitor ClickHouse query latency percentiles using system.query_log and Prometheus to track p50, p95, and p99 response times.

---

Monitoring query latency percentiles (p50, p95, p99) gives a much more accurate picture of user-perceived performance than average latency alone. A query that averages 100ms but has a p99 of 10 seconds indicates serious tail latency issues.

## Query Latency Percentiles from system.query_log

```sql
SELECT
    toStartOfMinute(query_start_time) AS minute,
    count() AS query_count,
    round(quantile(0.50)(query_duration_ms), 0) AS p50_ms,
    round(quantile(0.95)(query_duration_ms), 0) AS p95_ms,
    round(quantile(0.99)(query_duration_ms), 0) AS p99_ms,
    round(max(query_duration_ms), 0) AS max_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND query_start_time >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute DESC;
```

## Latency Percentiles by Table

```sql
SELECT
    tables[1] AS table_name,
    count() AS queries,
    round(quantile(0.50)(query_duration_ms), 0) AS p50_ms,
    round(quantile(0.95)(query_duration_ms), 0) AS p95_ms,
    round(quantile(0.99)(query_duration_ms), 0) AS p99_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND query_start_time >= now() - INTERVAL 24 HOUR
  AND length(tables) > 0
GROUP BY table_name
ORDER BY p99_ms DESC
LIMIT 20;
```

## Identify Slowest Queries

```sql
SELECT
    normalizeQuery(query) AS normalized_query,
    count() AS occurrences,
    round(avg(query_duration_ms), 0) AS avg_ms,
    round(quantile(0.99)(query_duration_ms), 0) AS p99_ms,
    round(max(query_duration_ms), 0) AS max_ms,
    formatReadableSize(avg(memory_usage)) AS avg_memory
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND query_start_time >= now() - INTERVAL 24 HOUR
GROUP BY normalized_query
ORDER BY p99_ms DESC
LIMIT 10;
```

## Track Latency SLO Compliance

```sql
-- Percentage of queries completing under 1 second (SLO target)
SELECT
    toStartOfHour(query_start_time) AS hour,
    count() AS total_queries,
    countIf(query_duration_ms <= 1000) AS under_1s,
    round(100.0 * countIf(query_duration_ms <= 1000) / count(), 2) AS slo_pct
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND query_start_time >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour DESC;
```

## Prometheus-Based Latency Monitoring

ClickHouse does not export query duration histograms natively in older versions. Use the HTTP interface to build a custom scraper, or rely on `system.query_log` via ClickHouse's built-in Prometheus endpoint with custom recording rules:

```yaml
# Prometheus recording rule
- record: clickhouse:query_duration_p99:5m
  expr: histogram_quantile(0.99, rate(clickhouse_query_duration_bucket[5m]))
```

## Alert on Latency Degradation

```sql
-- Query to detect sudden latency spikes (run as alerting query)
SELECT
    round(quantile(0.99)(query_duration_ms), 0) AS p99_last_5min
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND query_start_time >= now() - INTERVAL 5 MINUTE;
```

Alert if `p99_last_5min > 5000` (5 seconds).

## Latency Breakdown by Query Type

```sql
SELECT
    query_kind,
    round(quantile(0.50)(query_duration_ms), 0) AS p50_ms,
    round(quantile(0.99)(query_duration_ms), 0) AS p99_ms,
    count() AS count
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 1 HOUR
GROUP BY query_kind;
```

## Summary

Monitor ClickHouse query latency percentiles using `quantile()` aggregates over `system.query_log`. Track p50, p95, and p99 per minute for trending, per table to find problem tables, and against SLO targets to measure compliance. Alert on p99 spikes and use `normalizeQuery` to group similar queries when investigating root causes.
