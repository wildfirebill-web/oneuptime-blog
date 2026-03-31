# How to Track Deployment Performance Impact with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Deployment, Performance, Observability, CI/CD

Description: Track how deployments affect application performance in ClickHouse by correlating release events with latency and error rate changes.

---

Every deployment carries the risk of introducing regressions. ClickHouse lets you correlate deployment events with performance telemetry to detect regressions immediately after a release.

## Deployment Events Table

```sql
CREATE TABLE deployments
(
    deployment_id UUID DEFAULT generateUUIDv4(),
    service_name LowCardinality(String),
    environment LowCardinality(String),
    version LowCardinality(String),
    deployed_by String,
    deployed_at DateTime,
    rolled_back_at Nullable(DateTime) DEFAULT NULL
)
ENGINE = MergeTree()
ORDER BY (service_name, deployed_at);
```

## Performance Metrics Table

```sql
CREATE TABLE perf_snapshots
(
    service_name LowCardinality(String),
    version LowCardinality(String),
    p50_ms Float64,
    p95_ms Float64,
    p99_ms Float64,
    error_rate_pct Float64,
    rpm Float64,
    snapshot_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(snapshot_at)
ORDER BY (service_name, snapshot_at);
```

## Pre vs Post Deployment Latency Comparison

Compare p95 latency in the 1-hour window before and after each deployment.

```sql
WITH deployment AS (
    SELECT service_name, version, deployed_at
    FROM deployments
    WHERE service_name = 'checkout-service'
    ORDER BY deployed_at DESC
    LIMIT 1
)
SELECT
    'pre_deploy' AS window,
    round(quantile(0.95)(response_time_ms), 2) AS p95_ms,
    round(countIf(status_code >= 500) * 100.0 / count(), 3) AS error_pct,
    count() AS requests
FROM api_requests
JOIN deployment d ON api_requests.service_name = d.service_name
WHERE requested_at BETWEEN d.deployed_at - INTERVAL 1 HOUR AND d.deployed_at

UNION ALL

SELECT
    'post_deploy',
    round(quantile(0.95)(response_time_ms), 2),
    round(countIf(status_code >= 500) * 100.0 / count(), 3),
    count()
FROM api_requests
JOIN deployment d ON api_requests.service_name = d.service_name
WHERE requested_at BETWEEN d.deployed_at AND d.deployed_at + INTERVAL 1 HOUR;
```

## Version-to-Version Comparison

```sql
SELECT
    version,
    count() AS requests,
    round(quantile(0.50)(response_time_ms), 2) AS p50_ms,
    round(quantile(0.95)(response_time_ms), 2) AS p95_ms,
    round(quantile(0.99)(response_time_ms), 2) AS p99_ms,
    round(countIf(status_code >= 500) * 100.0 / count(), 3) AS error_rate_pct
FROM api_requests
WHERE service_name = 'checkout-service'
  AND requested_at >= now() - INTERVAL 7 DAY
GROUP BY version
ORDER BY min(requested_at);
```

## Rolling Error Rate Around Deployment Time

```sql
SELECT
    toStartOfFiveMinutes(requested_at) AS bucket,
    count() AS requests,
    countIf(status_code >= 500) AS errors,
    round(countIf(status_code >= 500) * 100.0 / count(), 3) AS error_pct
FROM api_requests
WHERE service_name = 'checkout-service'
  AND requested_at BETWEEN '2026-03-31 14:00:00' AND '2026-03-31 16:00:00'
GROUP BY bucket
ORDER BY bucket;
```

## Automatic Regression Detection

Find deployments where p99 latency increased by more than 20%.

```sql
WITH version_perf AS (
    SELECT
        version,
        min(snapshot_at) AS first_seen,
        avg(p99_ms) AS avg_p99
    FROM perf_snapshots
    WHERE service_name = 'checkout-service'
    GROUP BY version
),
ranked AS (
    SELECT
        version,
        first_seen,
        avg_p99,
        lagInFrame(avg_p99) OVER (ORDER BY first_seen) AS prev_p99
    FROM version_perf
    ORDER BY first_seen
)
SELECT
    version,
    first_seen,
    round(avg_p99, 2) AS p99_ms,
    round(prev_p99, 2) AS prev_p99_ms,
    round((avg_p99 - prev_p99) * 100.0 / prev_p99, 1) AS pct_change
FROM ranked
WHERE prev_p99 > 0 AND avg_p99 > prev_p99 * 1.2;
```

## Summary

ClickHouse makes deployment impact tracking straightforward by combining deployment event records with request log queries. Pre/post comparisons, version-by-version breakdowns, and automatic regression detection help engineering teams catch performance regressions within minutes of a release. This reduces mean time to detection without requiring a dedicated feature flagging or deployment intelligence platform.
