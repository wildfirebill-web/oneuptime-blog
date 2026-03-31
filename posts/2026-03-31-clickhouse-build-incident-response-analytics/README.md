# How to Build Incident Response Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Incident Response, MTTA, MTTR, On-Call Analytics

Description: Learn how to track incident timelines, MTTA, MTTR, and on-call load in ClickHouse to improve your incident response process.

---

Incident analytics drives process improvement. By storing alert, acknowledge, and resolution timestamps in ClickHouse, engineering teams can measure MTTA (Mean Time to Acknowledge), MTTR, recurring incident patterns, and on-call burden - then use this data to improve alerting and reduce toil.

## Schema

```sql
CREATE TABLE incidents
(
    incident_id   UInt64,
    service       LowCardinality(String),
    team          LowCardinality(String),
    severity      LowCardinality(String), -- 'P1','P2','P3','P4'
    created_at    DateTime,
    acked_at      Nullable(DateTime),
    resolved_at   Nullable(DateTime),
    root_cause    LowCardinality(String),
    responder_id  UInt64,
    is_after_hours UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (service, created_at);
```

## Mean Time to Acknowledge (MTTA)

```sql
SELECT
    severity,
    quantile(0.5)(dateDiff('minute', created_at, acked_at)) AS median_mtta_min,
    avg(dateDiff('minute', created_at, acked_at))           AS avg_mtta_min
FROM incidents
WHERE acked_at IS NOT NULL
  AND created_at >= now() - INTERVAL 90 DAY
GROUP BY severity
ORDER BY severity;
```

## Mean Time to Resolve (MTTR)

```sql
SELECT
    service,
    severity,
    quantile(0.5)(dateDiff('minute', created_at, resolved_at)) AS median_mttr_min,
    quantile(0.9)(dateDiff('minute', created_at, resolved_at)) AS p90_mttr_min
FROM incidents
WHERE resolved_at IS NOT NULL
  AND created_at >= now() - INTERVAL 90 DAY
GROUP BY service, severity
ORDER BY median_mttr_min DESC;
```

## Incident Volume by Week

```sql
SELECT
    toStartOfWeek(created_at) AS week,
    severity,
    count() AS incident_count
FROM incidents
WHERE created_at >= now() - INTERVAL 6 MONTH
GROUP BY week, severity
ORDER BY week, severity;
```

## Recurring Incidents by Root Cause

```sql
SELECT
    root_cause,
    count() AS incidents,
    sum(dateDiff('minute', created_at, resolved_at)) / 60 AS total_hours_lost
FROM incidents
WHERE created_at >= now() - INTERVAL 90 DAY
  AND resolved_at IS NOT NULL
GROUP BY root_cause
ORDER BY incidents DESC
LIMIT 20;
```

## After-Hours Incident Load

```sql
SELECT
    responder_id,
    count()                     AS total_incidents,
    countIf(is_after_hours = 1) AS after_hours_incidents,
    countIf(is_after_hours = 1) * 100.0 / count() AS after_hours_pct
FROM incidents
WHERE created_at >= now() - INTERVAL 90 DAY
GROUP BY responder_id
ORDER BY after_hours_incidents DESC
LIMIT 20;
```

## SLA Breach Rate

```sql
SELECT
    severity,
    count() AS total,
    countIf(
        dateDiff('minute', created_at, resolved_at) >
        multiIf(severity = 'P1', 60, severity = 'P2', 240, 1440)
    ) AS sla_breaches,
    sla_breaches * 100.0 / total AS breach_rate_pct
FROM incidents
WHERE resolved_at IS NOT NULL
  AND created_at >= now() - INTERVAL 90 DAY
GROUP BY severity;
```

## Summary

ClickHouse enables detailed incident analytics that drive meaningful process improvements. Compute MTTA, MTTR, and SLA breach rates by severity, identify recurring root causes, and measure after-hours on-call load per responder. These metrics give engineering leadership the data to justify staffing changes, automation investment, and reliability improvements.
