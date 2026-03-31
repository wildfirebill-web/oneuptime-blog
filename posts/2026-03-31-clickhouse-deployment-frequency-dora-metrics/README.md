# How to Track Deployment Frequency and DORA Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DORA Metric, Deployment, DevOps, Analytics

Description: Learn how to store and query DORA metrics including deployment frequency, lead time, and change failure rate using ClickHouse.

---

## Introduction to DORA Metrics

DORA (DevOps Research and Assessment) metrics measure engineering team performance. The four key metrics are deployment frequency, lead time for changes, change failure rate, and time to restore service. ClickHouse makes it easy to track all four at scale.

## Events Table

```sql
CREATE TABLE deployment_events (
    event_time DateTime,
    deploy_id String,
    service LowCardinality(String),
    environment LowCardinality(String),
    team LowCardinality(String),
    status LowCardinality(String),  -- success, failed, rolled_back
    commit_sha String,
    commit_time DateTime,
    duration_seconds UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (service, environment, event_time);
```

## Deployment Frequency

Count deployments to production per day per service:

```sql
SELECT
    toDate(event_time) AS day,
    service,
    countIf(status = 'success') AS successful_deploys,
    count() AS total_deploys
FROM deployment_events
WHERE environment = 'production'
  AND event_time >= now() - INTERVAL 30 DAY
GROUP BY day, service
ORDER BY day DESC, service;
```

## Lead Time for Changes

Measure the time from commit to production deployment:

```sql
SELECT
    service,
    avg(dateDiff('minute', commit_time, event_time)) AS avg_lead_time_minutes,
    quantile(0.95)(dateDiff('minute', commit_time, event_time)) AS p95_lead_time_minutes
FROM deployment_events
WHERE environment = 'production'
  AND status = 'success'
  AND event_time >= now() - INTERVAL 30 DAY
GROUP BY service
ORDER BY avg_lead_time_minutes DESC;
```

## Change Failure Rate

```sql
SELECT
    service,
    countIf(status IN ('failed', 'rolled_back')) * 100.0 / count() AS change_failure_rate_pct
FROM deployment_events
WHERE environment = 'production'
  AND event_time >= now() - INTERVAL 30 DAY
GROUP BY service
ORDER BY change_failure_rate_pct DESC;
```

## Time to Restore

Track incident recovery events alongside deployments:

```sql
CREATE TABLE incidents (
    incident_id String,
    service LowCardinality(String),
    environment LowCardinality(String),
    opened_at DateTime,
    resolved_at Nullable(DateTime)
) ENGINE = MergeTree()
ORDER BY (service, opened_at);

SELECT
    service,
    avg(dateDiff('minute', opened_at, resolved_at)) AS avg_restore_minutes
FROM incidents
WHERE environment = 'production'
  AND resolved_at IS NOT NULL
  AND opened_at >= now() - INTERVAL 30 DAY
GROUP BY service;
```

## DORA Summary Dashboard Query

```sql
SELECT
    service,
    countIf(status = 'success') AS deploy_freq_30d,
    avg(dateDiff('minute', commit_time, event_time)) AS avg_lead_time_min,
    countIf(status IN ('failed','rolled_back')) * 100.0 / count() AS failure_rate_pct
FROM deployment_events
WHERE environment = 'production'
  AND event_time >= now() - INTERVAL 30 DAY
GROUP BY service;
```

## Summary

ClickHouse makes it straightforward to compute DORA metrics from raw deployment event data. Using partitioned MergeTree tables and aggregation functions like `countIf` and `quantile`, you can build real-time dashboards that give engineering teams clear visibility into their delivery performance.
