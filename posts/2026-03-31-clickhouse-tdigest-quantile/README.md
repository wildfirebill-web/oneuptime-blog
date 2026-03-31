# How to Use t-digest for Quantile Estimation in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, t-digest, Quantile, Percentile, Approximate

Description: Learn how to use t-digest-based quantile functions in ClickHouse for accurate approximate percentile estimation on large datasets with low memory usage.

---

Computing exact percentiles requires sorting and storing all values, which is expensive at scale. ClickHouse implements the t-digest algorithm via `quantileTDigest` and related functions to estimate percentiles accurately with controlled memory usage.

## What Is t-digest

t-digest is a probabilistic data structure for estimating quantiles. It is highly accurate near the tails (p1, p99) and uses significantly less memory than exact methods. This makes it ideal for latency percentiles and response time analysis.

## Basic Quantile Estimation

```sql
SELECT
    quantileTDigest(0.5)(response_ms)  AS median,
    quantileTDigest(0.95)(response_ms) AS p95,
    quantileTDigest(0.99)(response_ms) AS p99
FROM api_requests
WHERE event_date = today();
```

## Multiple Quantiles at Once

```sql
SELECT
    quantilesTDigest(0.5, 0.75, 0.90, 0.95, 0.99)(response_ms) AS percentiles
FROM api_requests
WHERE event_date = today();
```

The result is an array: `[p50, p75, p90, p95, p99]`.

## Compression Parameter

The compression parameter controls the accuracy-memory tradeoff. Higher values give more accuracy at the cost of more memory:

```sql
SELECT quantileTDigest(200)(0.99)(response_ms) AS p99_accurate
FROM api_requests;
```

Default compression is 100. Values from 100 to 300 are common.

## Comparing t-digest with Exact Quantile

```sql
SELECT
    quantileExact(0.99)(response_ms)  AS p99_exact,
    quantileTDigest(0.99)(response_ms) AS p99_tdigest,
    abs(quantileExact(0.99)(response_ms) - quantileTDigest(0.99)(response_ms)) AS delta
FROM api_requests
WHERE event_date >= today() - 7;
```

## Pre-Aggregated t-digest States

Store t-digest states for rollups:

```sql
CREATE TABLE hourly_latency
(
    hour       DateTime,
    service    String,
    tdigest    AggregateFunction(quantileTDigest(100), Float64)
)
ENGINE = AggregatingMergeTree
ORDER BY (hour, service);

INSERT INTO hourly_latency
SELECT
    toStartOfHour(event_time) AS hour,
    service,
    quantileTDigestState(100)(toFloat64(response_ms)) AS tdigest
FROM api_requests
GROUP BY hour, service;

-- Query pre-aggregated data
SELECT
    service,
    quantileTDigestMerge(100)(0.99)(tdigest) AS p99
FROM hourly_latency
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY service
ORDER BY p99 DESC;
```

## p99 Latency by Endpoint

```sql
SELECT
    endpoint,
    count()                              AS requests,
    quantileTDigest(0.50)(response_ms)  AS p50_ms,
    quantileTDigest(0.95)(response_ms)  AS p95_ms,
    quantileTDigest(0.99)(response_ms)  AS p99_ms
FROM api_requests
WHERE event_date = today()
GROUP BY endpoint
ORDER BY p99_ms DESC
LIMIT 20;
```

## t-digest vs quantileExact vs quantileTiming

| Function | Accuracy | Memory | Speed |
|----------|----------|--------|-------|
| `quantileExact` | Exact | O(n) | Slow for large n |
| `quantileTDigest` | ~99% accurate at tails | O(compression) | Fast |
| `quantileTiming` | Fixed bins, less accurate | O(1) | Fastest |

## Summary

ClickHouse's `quantileTDigest` provides accurate approximate percentile estimation using the t-digest algorithm. With default compression of 100, it uses kilobytes of memory per group while achieving ~99% accuracy at tail percentiles. It is the recommended approach for latency percentile monitoring on high-volume datasets where exact quantiles would be prohibitively expensive.
