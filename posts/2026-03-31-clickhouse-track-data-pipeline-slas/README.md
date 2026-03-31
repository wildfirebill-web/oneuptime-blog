# How to Track Data Pipeline SLAs with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SLA, Data Pipeline, Monitoring, Freshness, Reliability

Description: Track data pipeline SLAs in ClickHouse by measuring latency, completeness, and freshness metrics, and alerting when pipelines breach defined service level objectives.

---

## Defining Data Pipeline SLAs

A data pipeline SLA defines the expected behavior of your data flow:
- **Freshness SLA** - Data must arrive within X minutes of the source event
- **Completeness SLA** - At least Y% of expected rows must arrive per hour
- **Latency SLA** - p99 ingestion-to-query latency must be under Z seconds
- **Availability SLA** - Pipeline must successfully deliver data for W% of time intervals

## Tracking Freshness SLA

```sql
CREATE TABLE pipeline_sla_log (
    check_time    DateTime DEFAULT now(),
    pipeline_name LowCardinality(String),
    data_source   LowCardinality(String),
    lag_seconds   UInt32,
    sla_threshold UInt32,
    is_breached   UInt8
) ENGINE = MergeTree()
ORDER BY (pipeline_name, check_time);
```

Populate on a schedule (e.g., every minute):

```sql
INSERT INTO pipeline_sla_log
SELECT
    now(),
    'events_pipeline',
    'kafka_consumer',
    dateDiff('second', max(event_time), now()) AS lag_seconds,
    300 AS sla_threshold,
    IF(dateDiff('second', max(event_time), now()) > 300, 1, 0) AS is_breached
FROM events;
```

## Measuring SLA Compliance Rate

```sql
SELECT
    pipeline_name,
    toDate(check_time) AS day,
    count() AS total_checks,
    countIf(is_breached = 0) AS successful_checks,
    countIf(is_breached = 0) * 100.0 / count() AS compliance_pct
FROM pipeline_sla_log
WHERE check_time >= today() - 30
GROUP BY pipeline_name, day
ORDER BY day DESC, pipeline_name;
```

## Completeness SLA

Track expected vs actual row counts per hour:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS received_rows,
    IF(count() >= 10000, 'OK', 'BREACH') AS completeness_status
FROM events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour DESC;
```

Combine with an expected-count table from your source system for precise tracking.

## End-to-End Latency SLA

If you store both event time and ingestion time:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    avg(dateDiff('second', event_time, ingested_at)) AS avg_latency_seconds,
    quantile(0.99)(dateDiff('second', event_time, ingested_at)) AS p99_latency_seconds
FROM events
WHERE event_time >= today()
GROUP BY hour
ORDER BY hour DESC;
```

Alert when `p99_latency_seconds > your_sla_threshold`.

## Monthly SLA Report

```sql
SELECT
    pipeline_name,
    toYYYYMM(check_time) AS month,
    count() AS total_checks,
    countIf(is_breached = 0) * 100.0 / count() AS uptime_pct,
    sum(lag_seconds * is_breached) AS total_breach_seconds
FROM pipeline_sla_log
WHERE check_time >= toStartOfMonth(now() - INTERVAL 3 MONTH)
GROUP BY pipeline_name, month
ORDER BY pipeline_name, month DESC;
```

## Summary

Track data pipeline SLAs in ClickHouse by storing freshness, completeness, and latency measurements in a dedicated SLA log table, computing compliance rates over rolling windows, and generating monthly reports. Alert on SLA breaches by querying the log table from your monitoring system and triggering notifications when compliance drops below your agreed thresholds.
