# How to Build Incident Response Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Incident Response, Analytics, SRE, Monitoring

Description: Learn how to build incident response analytics in ClickHouse to track MTTR, incident frequency, severity trends, and team response patterns.

---

## Why Incident Analytics Matter

Understanding your incident history is key to improving reliability. ClickHouse lets you store and query incident data alongside logs, metrics, and traces - giving you a unified view of your systems health.

## Incidents Table

```sql
CREATE TABLE incidents (
    incident_id String,
    title String,
    service LowCardinality(String),
    severity LowCardinality(String),  -- P1, P2, P3, P4
    status LowCardinality(String),    -- open, resolved, closed
    opened_at DateTime,
    acknowledged_at Nullable(DateTime),
    resolved_at Nullable(DateTime),
    team LowCardinality(String),
    root_cause LowCardinality(String),
    affected_users UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(opened_at)
ORDER BY (service, severity, opened_at);
```

## Incident Frequency by Service

```sql
SELECT
    service,
    severity,
    count() AS incident_count,
    toStartOfWeek(opened_at) AS week
FROM incidents
WHERE opened_at >= now() - INTERVAL 90 DAY
GROUP BY service, severity, week
ORDER BY week DESC, incident_count DESC;
```

## MTTR (Mean Time to Resolution)

```sql
SELECT
    service,
    severity,
    avg(dateDiff('minute', opened_at, resolved_at)) AS avg_mttr_minutes,
    quantile(0.95)(dateDiff('minute', opened_at, resolved_at)) AS p95_mttr_minutes,
    count() AS incident_count
FROM incidents
WHERE resolved_at IS NOT NULL
  AND opened_at >= now() - INTERVAL 30 DAY
GROUP BY service, severity
ORDER BY avg_mttr_minutes DESC;
```

## Time to Acknowledge

```sql
SELECT
    team,
    severity,
    avg(dateDiff('minute', opened_at, acknowledged_at)) AS avg_tta_minutes
FROM incidents
WHERE acknowledged_at IS NOT NULL
  AND opened_at >= now() - INTERVAL 30 DAY
GROUP BY team, severity
ORDER BY severity, avg_tta_minutes DESC;
```

## Root Cause Distribution

```sql
SELECT
    root_cause,
    count() AS count,
    count() * 100.0 / sum(count()) OVER () AS pct
FROM incidents
WHERE opened_at >= now() - INTERVAL 90 DAY
GROUP BY root_cause
ORDER BY count DESC;
```

## Affected Users Impact

```sql
SELECT
    toDate(opened_at) AS day,
    sum(affected_users) AS total_affected,
    countIf(severity = 'P1') AS p1_incidents,
    countIf(severity = 'P2') AS p2_incidents
FROM incidents
WHERE opened_at >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day DESC;
```

## Recurrence Detection

Find services with repeated incidents from the same root cause:

```sql
SELECT
    service,
    root_cause,
    count() AS recurrence_count,
    min(opened_at) AS first_seen,
    max(opened_at) AS last_seen
FROM incidents
WHERE opened_at >= now() - INTERVAL 90 DAY
GROUP BY service, root_cause
HAVING recurrence_count > 2
ORDER BY recurrence_count DESC;
```

## Summary

ClickHouse is an excellent foundation for incident analytics. By storing incident lifecycle events in a MergeTree table, you can track MTTR trends, identify repeat offenders, measure team response times, and report on reliability improvements - all with sub-second query performance.
