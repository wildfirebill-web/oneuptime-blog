# How to Calculate Apdex Score in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apdex, Performance, SLO, Observability, Analytics

Description: Learn how to calculate Apdex score in ClickHouse to measure user satisfaction with application response times using SQL aggregations.

---

Apdex (Application Performance Index) is a standardized metric for measuring user satisfaction based on response times. It gives you a single number between 0 and 1 that reflects how well your application is performing from a user perspective.

## Apdex Formula

The Apdex score is calculated as:

```text
Apdex = (Satisfied + Tolerating / 2) / Total

Where:
- Satisfied:  response_time <= T
- Tolerating: response_time > T AND response_time <= 4T
- Frustrated: response_time > 4T
```

A common threshold T is 500ms for web applications.

## Schema Setup

```sql
CREATE TABLE http_requests (
    timestamp DateTime,
    service String,
    endpoint String,
    duration_ms UInt32
) ENGINE = MergeTree()
ORDER BY (service, timestamp);
```

## Basic Apdex Calculation

With T = 500ms:

```sql
WITH 500 AS t
SELECT
    service,
    toStartOfHour(timestamp) AS hour,
    countIf(duration_ms <= t) AS satisfied,
    countIf(duration_ms > t AND duration_ms <= t * 4) AS tolerating,
    countIf(duration_ms > t * 4) AS frustrated,
    count() AS total,
    round((countIf(duration_ms <= t) + countIf(duration_ms > t AND duration_ms <= t * 4) / 2.0) / count(), 4) AS apdex
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY service, hour
ORDER BY service, hour;
```

## Per-Endpoint Apdex

Different endpoints may warrant different thresholds. You can use a CASE expression:

```sql
SELECT
    endpoint,
    countIf(
        duration_ms <= multiIf(
            endpoint = '/api/search', 1000,
            endpoint = '/api/login', 200,
            500
        )
    ) AS satisfied,
    count() AS total,
    round(
        (
            countIf(duration_ms <= multiIf(endpoint = '/api/search', 1000, endpoint = '/api/login', 200, 500)) +
            countIf(duration_ms > multiIf(endpoint = '/api/search', 1000, endpoint = '/api/login', 200, 500)
                AND duration_ms <= multiIf(endpoint = '/api/search', 4000, endpoint = '/api/login', 800, 2000)
            ) / 2.0
        ) / count(),
        4
    ) AS apdex
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY endpoint
ORDER BY apdex ASC;
```

## Materialized View for Continuous Tracking

Pre-aggregate counts for performance:

```sql
CREATE MATERIALIZED VIEW apdex_mv
ENGINE = SummingMergeTree()
ORDER BY (service, hour)
AS
SELECT
    service,
    toStartOfHour(timestamp) AS hour,
    countIf(duration_ms <= 500) AS satisfied,
    countIf(duration_ms > 500 AND duration_ms <= 2000) AS tolerating,
    count() AS total
FROM http_requests
GROUP BY service, hour;
```

Query the view:

```sql
SELECT
    service,
    hour,
    round((sum(satisfied) + sum(tolerating) / 2.0) / sum(total), 4) AS apdex
FROM apdex_mv
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY service, hour
ORDER BY service, hour;
```

## Interpreting Apdex Scores

```text
1.0       - Excellent
0.94-1.0  - Excellent
0.85-0.94 - Good
0.70-0.85 - Fair
0.50-0.70 - Poor
< 0.50    - Unacceptable
```

## Summary

ClickHouse provides all the aggregation primitives needed to compute Apdex scores efficiently. By using `countIf` with threshold comparisons and materialized views for continuous pre-aggregation, you can build real-time Apdex dashboards that scale to millions of requests per second.
