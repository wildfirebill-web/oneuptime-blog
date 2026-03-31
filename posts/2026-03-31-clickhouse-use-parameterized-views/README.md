# How to Use Parameterized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Parameterized View, View, SQL, Query Reuse, Dynamic Query

Description: Learn how to create and use parameterized views in ClickHouse to build reusable query templates that accept runtime parameters.

---

Parameterized views in ClickHouse are views that accept query parameters, allowing you to reuse complex query logic with different inputs at runtime. This combines the reusability of views with the flexibility of parameterized queries.

## Creating a Parameterized View

Use `CREATE VIEW` with `{param_name:type}` placeholders:

```sql
CREATE VIEW service_metrics AS
SELECT
    service,
    toStartOfInterval(timestamp, INTERVAL {interval:UInt32} SECOND) AS bucket,
    count() AS requests,
    countIf(status_code >= 500) AS errors,
    avg(duration_ms) AS avg_latency
FROM http_requests
WHERE timestamp >= now() - INTERVAL {lookback_hours:UInt32} HOUR
  AND service = {service_name:String}
GROUP BY service, bucket
ORDER BY bucket;
```

## Querying a Parameterized View

Pass parameters using the URL parameter syntax:

```sql
SELECT * FROM service_metrics(
    interval = 3600,
    lookback_hours = 24,
    service_name = 'api-gateway'
);
```

## More Complex Example: Apdex by Service

```sql
CREATE VIEW apdex_view AS
SELECT
    service,
    toStartOfHour(timestamp) AS hour,
    countIf(duration_ms <= {threshold_ms:UInt32}) AS satisfied,
    countIf(duration_ms > {threshold_ms:UInt32} AND duration_ms <= {threshold_ms:UInt32} * 4) AS tolerating,
    count() AS total,
    round(
        (countIf(duration_ms <= {threshold_ms:UInt32}) + countIf(duration_ms > {threshold_ms:UInt32} AND duration_ms <= {threshold_ms:UInt32} * 4) / 2.0)
        / count(), 4
    ) AS apdex_score
FROM http_requests
WHERE timestamp >= now() - INTERVAL {hours:UInt32} HOUR
GROUP BY service, hour
ORDER BY service, hour;
```

Query it with different thresholds:

```sql
-- 500ms threshold
SELECT * FROM apdex_view(threshold_ms = 500, hours = 24);

-- Stricter 200ms threshold
SELECT * FROM apdex_view(threshold_ms = 200, hours = 6);
```

## Supported Parameter Types

```text
UInt8, UInt16, UInt32, UInt64
Int8, Int16, Int32, Int64
Float32, Float64
String
Date, DateTime
Array(T)
```

## Using Parameters in Subqueries and CTEs

Parameters propagate into nested expressions:

```sql
CREATE VIEW top_users AS
WITH
    base AS (
        SELECT user_id, count() AS events
        FROM events
        WHERE timestamp >= now() - INTERVAL {days:UInt32} DAY
          AND event_type = {event_type:String}
        GROUP BY user_id
    )
SELECT user_id, events
FROM base
WHERE events >= {min_events:UInt32}
ORDER BY events DESC
LIMIT {top_n:UInt32};
```

```sql
SELECT * FROM top_users(days = 7, event_type = 'purchase', min_events = 5, top_n = 100);
```

## Dropping a Parameterized View

```sql
DROP VIEW IF EXISTS service_metrics;
```

## Summary

Parameterized views in ClickHouse let you encapsulate complex query logic into reusable templates with type-safe runtime parameters. They are ideal for dashboard queries, API backends, and analytical frameworks where the same query shape is run with different filters, time windows, or thresholds.
