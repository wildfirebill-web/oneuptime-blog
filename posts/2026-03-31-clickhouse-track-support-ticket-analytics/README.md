# How to Track Support Ticket Analytics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Support Analytics, Ticket, SLA, Resolution Time

Description: Learn how to track support ticket volumes, SLA compliance, resolution times, and agent performance in ClickHouse.

---

Support teams need visibility into ticket volumes, response times, and SLA compliance. ClickHouse handles the event stream from ticketing systems and enables fast aggregations over historical ticket data, so support managers can track trends and identify bottlenecks without waiting for nightly reports.

## Schema

```sql
CREATE TABLE support_tickets
(
    ticket_id       UInt64,
    created_at      DateTime,
    first_response  DateTime,
    resolved_at     Nullable(DateTime),
    account_id      UInt64,
    plan            LowCardinality(String),
    priority        LowCardinality(String), -- 'low','medium','high','critical'
    category        LowCardinality(String),
    agent_id        UInt64,
    status          LowCardinality(String),
    csat_score      Nullable(UInt8)         -- 1-5
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, ticket_id);
```

## Ticket Volume by Day

```sql
SELECT
    toDate(created_at) AS day,
    priority,
    count() AS tickets
FROM support_tickets
WHERE created_at >= now() - INTERVAL 30 DAY
GROUP BY day, priority
ORDER BY day, priority;
```

## First Response Time - SLA Compliance

Most SLAs require a first response within 1 hour for critical tickets:

```sql
SELECT
    priority,
    count() AS total,
    countIf(dateDiff('minute', created_at, first_response) <= 60) AS within_sla,
    within_sla * 100.0 / total AS sla_compliance_pct,
    quantile(0.5)(dateDiff('minute', created_at, first_response)) AS median_frt_min
FROM support_tickets
WHERE created_at >= now() - INTERVAL 30 DAY
GROUP BY priority
ORDER BY priority;
```

## Average Resolution Time by Category

```sql
SELECT
    category,
    quantile(0.5)(dateDiff('hour', created_at, resolved_at)) AS median_resolution_hr,
    quantile(0.9)(dateDiff('hour', created_at, resolved_at)) AS p90_resolution_hr
FROM support_tickets
WHERE resolved_at IS NOT NULL
  AND created_at >= now() - INTERVAL 90 DAY
GROUP BY category
ORDER BY median_resolution_hr DESC;
```

## Agent Performance

```sql
SELECT
    agent_id,
    count()             AS tickets_handled,
    avg(csat_score)     AS avg_csat,
    quantile(0.5)(dateDiff('hour', created_at, resolved_at)) AS median_resolve_hr
FROM support_tickets
WHERE resolved_at IS NOT NULL
  AND created_at >= now() - INTERVAL 30 DAY
GROUP BY agent_id
ORDER BY avg_csat DESC
LIMIT 20;
```

## CSAT Trend by Month

```sql
SELECT
    toStartOfMonth(created_at) AS month,
    avg(csat_score)            AS avg_csat,
    count()                    AS rated_tickets
FROM support_tickets
WHERE csat_score IS NOT NULL
GROUP BY month
ORDER BY month;
```

## High-Priority Ticket Backlog

Open critical/high tickets older than 4 hours:

```sql
SELECT
    ticket_id,
    created_at,
    dateDiff('hour', created_at, now()) AS age_hours,
    account_id,
    plan
FROM support_tickets
WHERE status = 'open'
  AND priority IN ('critical', 'high')
  AND dateDiff('hour', created_at, now()) > 4
ORDER BY age_hours DESC;
```

## Summary

ClickHouse supports comprehensive support analytics by storing ticket events and computing SLA compliance, resolution times, agent performance, and CSAT trends. Query in real time across millions of historical tickets without waiting for daily batch jobs, giving support managers the visibility they need to staff appropriately and meet service commitments.
