# How to Track CI/CD Pipeline Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CI/CD, Pipeline, Build Time, DORA Metric

Description: Learn how to store CI/CD pipeline events in ClickHouse and query build duration, failure rates, and pipeline throughput trends.

---

CI/CD pipelines generate structured events - pipeline starts, stage completions, test results, and deployment outcomes. Centralizing these in ClickHouse lets engineering managers measure delivery performance, catch flaky tests, and track trends toward DORA metric targets.

## Schema

```sql
CREATE TABLE pipeline_runs
(
    run_id       UInt64,
    ts_start     DateTime,
    ts_end       Nullable(DateTime),
    repo         LowCardinality(String),
    branch       LowCardinality(String),
    stage        LowCardinality(String), -- 'build','test','lint','deploy'
    status       LowCardinality(String), -- 'success','failure','cancelled','timeout'
    triggered_by LowCardinality(String), -- 'push','pr','schedule','manual'
    runner       LowCardinality(String),
    duration_s   UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts_start)
ORDER BY (repo, ts_start);
```

## Build Duration Percentiles by Repo

```sql
SELECT
    repo,
    stage,
    quantile(0.5)(duration_s) / 60  AS median_min,
    quantile(0.95)(duration_s) / 60 AS p95_min
FROM pipeline_runs
WHERE ts_start >= now() - INTERVAL 30 DAY
  AND status = 'success'
GROUP BY repo, stage
ORDER BY repo, median_min DESC;
```

## Failure Rate by Stage

```sql
SELECT
    stage,
    countIf(status = 'failure') * 100.0 / count() AS failure_rate_pct,
    count() AS total_runs
FROM pipeline_runs
WHERE ts_start >= now() - INTERVAL 7 DAY
GROUP BY stage
ORDER BY failure_rate_pct DESC;
```

## Pipeline Throughput per Day

```sql
SELECT
    toDate(ts_start) AS day,
    count()          AS total_runs,
    countIf(status = 'success') AS successful
FROM pipeline_runs
WHERE ts_start >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day;
```

## Flaky Stages (High Variance in Duration)

```sql
SELECT
    repo,
    stage,
    stddevPop(duration_s) AS stddev_s,
    avg(duration_s)       AS avg_s,
    stddevPop(duration_s) / avg(duration_s) AS cv
FROM pipeline_runs
WHERE ts_start >= now() - INTERVAL 30 DAY
  AND status = 'success'
GROUP BY repo, stage
HAVING avg_s > 60
ORDER BY cv DESC
LIMIT 20;
```

## Mean Time to Recovery (MTTR)

How long between a failed build and the next successful one?

```sql
SELECT
    repo,
    avg(dateDiff('minute', fail_ts, recover_ts)) AS avg_mttr_min
FROM (
    SELECT
        repo,
        ts_start AS fail_ts,
        min(ts_start) OVER (
            PARTITION BY repo
            ORDER BY ts_start
            ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
        ) AS recover_ts,
        status
    FROM pipeline_runs
    WHERE ts_start >= now() - INTERVAL 30 DAY
)
WHERE status = 'failure'
GROUP BY repo
ORDER BY avg_mttr_min DESC;
```

## Summary

ClickHouse enables robust CI/CD analytics by storing pipeline run events and providing fast aggregation over them. Measure build duration trends, failure rates by stage, pipeline throughput, and MTTR - all key DORA metrics. These insights help engineering teams find slow builds, fix flaky stages, and improve overall delivery performance.
