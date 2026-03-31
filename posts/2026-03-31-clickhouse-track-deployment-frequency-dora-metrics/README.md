# How to Track Deployment Frequency and DORA Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DORA Metric, Deployment Frequency, Change Failure Rate, Lead Time

Description: Learn how to store deployment events in ClickHouse and compute the four DORA metrics - deployment frequency, lead time, MTTR, and change failure rate.

---

DORA metrics (Deployment Frequency, Lead Time for Changes, Mean Time to Restore, and Change Failure Rate) are the standard way to measure engineering team performance. ClickHouse makes it easy to compute all four from deployment and incident event data.

## Schema

```sql
CREATE TABLE deployments
(
    deploy_id    UInt64,
    deployed_at  DateTime,
    service      LowCardinality(String),
    team         LowCardinality(String),
    environment  LowCardinality(String),
    commit_sha   FixedString(40),
    commit_time  DateTime,
    status       LowCardinality(String), -- 'success','failure','rollback'
    version      LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(deployed_at)
ORDER BY (service, deployed_at);

CREATE TABLE incidents
(
    incident_id  UInt64,
    started_at   DateTime,
    resolved_at  Nullable(DateTime),
    service      LowCardinality(String),
    deploy_id    Nullable(UInt64),
    severity     LowCardinality(String)
)
ENGINE = MergeTree()
ORDER BY (service, started_at);
```

## 1. Deployment Frequency (per Day)

```sql
SELECT
    toDate(deployed_at) AS day,
    service,
    countIf(status = 'success') AS deployments
FROM deployments
WHERE environment = 'production'
  AND deployed_at >= now() - INTERVAL 30 DAY
GROUP BY day, service
ORDER BY day, service;
```

## 2. Lead Time for Changes

Time from commit to production deployment:

```sql
SELECT
    service,
    quantile(0.5)(dateDiff('hour', commit_time, deployed_at)) AS median_lead_time_hr,
    quantile(0.9)(dateDiff('hour', commit_time, deployed_at)) AS p90_lead_time_hr
FROM deployments
WHERE environment = 'production'
  AND status = 'success'
  AND deployed_at >= now() - INTERVAL 30 DAY
GROUP BY service
ORDER BY median_lead_time_hr;
```

## 3. Change Failure Rate

```sql
SELECT
    service,
    countIf(status IN ('failure', 'rollback')) * 100.0 / count() AS change_failure_rate_pct
FROM deployments
WHERE environment = 'production'
  AND deployed_at >= now() - INTERVAL 30 DAY
GROUP BY service
ORDER BY change_failure_rate_pct DESC;
```

## 4. Mean Time to Restore (MTTR)

```sql
SELECT
    i.service,
    avg(dateDiff('minute', i.started_at, i.resolved_at)) AS avg_mttr_min
FROM incidents i
WHERE i.resolved_at IS NOT NULL
  AND i.started_at >= now() - INTERVAL 30 DAY
GROUP BY i.service
ORDER BY avg_mttr_min DESC;
```

## DORA Level Classification

Classify each service by DORA performance level:

```sql
SELECT
    service,
    deployments_per_day,
    multiIf(
        deployments_per_day >= 1,   'Elite',
        deployments_per_day >= 0.14,'High',   -- weekly
        deployments_per_day >= 0.03,'Medium', -- monthly
        'Low'
    ) AS dora_level
FROM (
    SELECT
        service,
        countIf(status = 'success') / 30.0 AS deployments_per_day
    FROM deployments
    WHERE environment = 'production'
      AND deployed_at >= now() - INTERVAL 30 DAY
    GROUP BY service
)
ORDER BY deployments_per_day DESC;
```

## Summary

ClickHouse enables straightforward DORA metric computation from deployment and incident event data. Track deployment frequency, lead time, change failure rate, and MTTR with simple SQL aggregations. Classify teams by DORA performance level and track improvement over time to drive engineering excellence conversations backed by real data.
