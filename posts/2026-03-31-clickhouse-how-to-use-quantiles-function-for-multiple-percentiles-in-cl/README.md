# How to Use quantiles() Function for Multiple Percentiles in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Percentiles, Quantiles, Analytics, Performance

Description: Learn how to use the quantiles() function in ClickHouse to compute multiple percentiles in a single query pass for efficient latency and metric analysis.

---

## Overview

ClickHouse's `quantiles()` function computes multiple quantiles (percentiles) in a single aggregation pass. This is far more efficient than running separate `quantile()` calls because the data is only scanned once.

## Basic Syntax

```sql
SELECT quantiles(0.5, 0.9, 0.95, 0.99)(response_time_ms)
FROM request_logs;
```

The function returns an array of quantile values corresponding to the levels provided.

## Unpacking the Result Array

To work with individual percentile values, use array indexing:

```sql
SELECT
    quantiles(0.5, 0.9, 0.95, 0.99)(response_time_ms) AS q,
    q[1] AS p50,
    q[2] AS p90,
    q[3] AS p95,
    q[4] AS p99
FROM request_logs;
```

## Grouping by Endpoint

```sql
SELECT
    endpoint,
    quantiles(0.5, 0.95, 0.99)(response_time_ms) AS q,
    round(q[1], 2) AS p50_ms,
    round(q[2], 2) AS p95_ms,
    round(q[3], 2) AS p99_ms
FROM request_logs
GROUP BY endpoint
ORDER BY p99_ms DESC;
```

## Time-Based Percentile Analysis

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    quantiles(0.5, 0.9, 0.99)(duration_ms) AS q,
    q[1] AS p50,
    q[2] AS p90,
    q[3] AS p99
FROM api_events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;
```

## Difference Between quantile() and quantiles()

```sql
-- Less efficient: 3 separate scans
SELECT
    quantile(0.5)(val),
    quantile(0.95)(val),
    quantile(0.99)(val)
FROM metrics;

-- More efficient: single scan
SELECT
    quantiles(0.5, 0.95, 0.99)(val)
FROM metrics;
```

## Using quantilesExact() for Small Datasets

For small datasets where exact values matter, use `quantilesExact()`:

```sql
SELECT quantilesExact(0.5, 0.95, 0.99)(response_time_ms)
FROM request_logs
WHERE service = 'checkout'
LIMIT 10000;
```

## Combining with WITH for Readable Output

```sql
WITH quantiles(0.5, 0.75, 0.9, 0.95, 0.99)(duration_ms) AS q
SELECT
    q[1] AS p50,
    q[2] AS p75,
    q[3] AS p90,
    q[4] AS p95,
    q[5] AS p99
FROM events;
```

## Performance Considerations

- `quantiles()` uses reservoir sampling by default (approximate but fast).
- Use `quantilesExact()` when you need precise values (higher memory usage).
- The function works with all numeric types.
- It integrates with the `-If` combinator: `quantilesIf(0.5, 0.99)(val, condition)`.

```sql
SELECT
    quantilesIf(0.5, 0.99)(response_ms, status = 200) AS success_q,
    quantilesIf(0.5, 0.99)(response_ms, status >= 500) AS error_q
FROM api_logs;
```

## Summary

The `quantiles()` function is the most efficient way to compute multiple percentiles in ClickHouse because it aggregates in a single data pass. It returns an array indexed by position, and it pairs well with `WITH` clauses and the `-If` combinator for conditional analysis. Use it for latency distributions, SLO tracking, and performance dashboards.
