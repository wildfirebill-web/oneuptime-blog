# How to Use quantiles() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Quantile, Percentile

Description: Learn how to compute multiple quantiles in a single pass in ClickHouse using quantiles(), which is more efficient than calling quantile() repeatedly.

---

`quantiles()` is a ClickHouse aggregate function that computes multiple quantile levels over a column in a single aggregation pass, returning an array of results. When you need p50, p90, p95, and p99 simultaneously - which is the norm in latency dashboards and SLO reports - `quantiles()` is the correct and most efficient tool. Calling `quantile()` separately for each level is functionally equivalent but semantically redundant and harder to read.

## Syntax

```sql
quantiles(level1, level2, ...)(column)
```

All levels are Float64 constants between 0 and 1. The function returns an Array(Float64) where the element at index `i` corresponds to `level_i`. Levels do not need to be sorted in the call, but the output array preserves the order in which you listed the levels.

## Basic Example

```sql
-- Compute p50, p90, p95, p99 in a single pass
SELECT
    quantiles(0.5, 0.9, 0.95, 0.99)(response_time_ms) AS percentiles
FROM api_requests
WHERE request_date = today() - 1;
-- Returns: [p50_value, p90_value, p95_value, p99_value]
```

## Accessing Individual Elements

Use array index notation (1-based in ClickHouse) to extract a specific level from the result.

```sql
SELECT
    endpoint,
    quantiles(0.5, 0.9, 0.99)(response_time_ms)[1] AS p50,
    quantiles(0.5, 0.9, 0.99)(response_time_ms)[2] AS p90,
    quantiles(0.5, 0.9, 0.99)(response_time_ms)[3] AS p99
FROM api_requests
WHERE request_date = today() - 1
GROUP BY endpoint
ORDER BY p99 DESC;
```

## Grouped Latency Percentiles

```sql
-- Service-level latency percentile report for the past 7 days
SELECT
    service_name,
    region,
    count()                                              AS total_requests,
    quantiles(0.5, 0.75, 0.9, 0.95, 0.99)(latency_ms)  AS pcts
FROM service_traces
WHERE trace_date >= today() - 7
GROUP BY service_name, region
ORDER BY pcts[4] DESC;  -- sort by p95
```

## Expanding the Array into Named Columns

For readability in downstream tooling, use array index accessors to give each percentile a name.

```sql
SELECT
    service_name,
    region,
    pcts[1] AS p50,
    pcts[2] AS p75,
    pcts[3] AS p90,
    pcts[4] AS p95,
    pcts[5] AS p99
FROM (
    SELECT
        service_name,
        region,
        quantiles(0.5, 0.75, 0.9, 0.95, 0.99)(latency_ms) AS pcts
    FROM service_traces
    WHERE trace_date >= today() - 7
    GROUP BY service_name, region
)
ORDER BY p99 DESC
LIMIT 50;
```

## Efficiency vs Multiple quantile() Calls

Both of the following queries produce the same result, but the `quantiles()` form is preferred.

```sql
-- Less idiomatic: separate calls (ClickHouse may still optimize into one pass)
SELECT
    quantile(0.5)(latency_ms)  AS p50,
    quantile(0.9)(latency_ms)  AS p90,
    quantile(0.99)(latency_ms) AS p99
FROM api_requests
WHERE request_date = today() - 1;

-- Preferred: single explicit call
SELECT
    quantiles(0.5, 0.9, 0.99)(latency_ms) AS pcts
FROM api_requests
WHERE request_date = today() - 1;
```

Using `quantiles()` makes the single-pass intent explicit and is the canonical form in ClickHouse documentation.

## SLO Dashboard Query

```sql
-- Complete SLO report: request volume, error rate, and latency percentiles
SELECT
    endpoint,
    count()                                                AS total_requests,
    countIf(status_code >= 500) * 100.0 / count()         AS error_rate_pct,
    quantiles(0.5, 0.9, 0.99)(latency_ms)[1]              AS p50_ms,
    quantiles(0.5, 0.9, 0.99)(latency_ms)[2]              AS p90_ms,
    quantiles(0.5, 0.9, 0.99)(latency_ms)[3]              AS p99_ms,
    quantiles(0.5, 0.9, 0.99)(latency_ms)[3] > 500        AS p99_slo_breach
FROM api_requests
WHERE request_time >= now() - INTERVAL 1 HOUR
GROUP BY endpoint
ORDER BY p99_ms DESC;
```

## quantilesExact and Other Variants

Like `quantile()`, `quantiles()` has exact and algorithm-specific variants.

```sql
-- Exact quantiles (sorts all data - only for small groups)
SELECT quantilesExact(0.5, 0.9, 0.99)(latency_ms) AS exact_pcts
FROM api_requests
WHERE request_date = today() AND endpoint = '/health';

-- t-digest variant for better tail accuracy
SELECT quantilesTDigest(0.9, 0.95, 0.99)(latency_ms) AS tdigest_pcts
FROM api_requests
WHERE request_date = today() - 1;
```

## Pre-aggregating Quantiles with AggregatingMergeTree

```sql
CREATE TABLE hourly_latency_agg
(
    service_name String,
    event_hour   DateTime,
    latency_pcts AggregateFunction(quantiles(0.5, 0.9, 0.99), UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (service_name, event_hour);

-- Insert state
INSERT INTO hourly_latency_agg
SELECT
    service_name,
    toStartOfHour(request_time) AS event_hour,
    quantilesState(0.5, 0.9, 0.99)(latency_ms) AS latency_pcts
FROM api_requests
GROUP BY service_name, event_hour;

-- Query with merge
SELECT
    service_name,
    event_hour,
    quantilesMerge(0.5, 0.9, 0.99)(latency_pcts) AS pcts
FROM hourly_latency_agg
WHERE event_hour >= now() - INTERVAL 24 HOUR
GROUP BY service_name, event_hour
ORDER BY service_name, event_hour;
```

## Summary

`quantiles(level1, level2, ...)(col)` is the most efficient way to compute multiple percentiles simultaneously in ClickHouse, returning a single array from one aggregation pass. Use array indexing to extract named columns, and prefer this function over repeated `quantile()` calls whenever you need more than one percentile level. For pre-aggregated dashboards, the `-State` and `-Merge` combinators with `AggregatingMergeTree` make quantile queries sub-second even at massive scale.
