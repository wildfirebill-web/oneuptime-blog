# How to Build a Serving Layer with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Serving Layer, API, Low Latency, OLAP

Description: Build a high-performance serving layer with ClickHouse to power user-facing dashboards and APIs with sub-second query latency at scale.

---

## What Is a Serving Layer

The serving layer is the component that answers queries from applications, dashboards, and APIs. It sits between raw data storage and end users, and must return results quickly - typically under 100ms for user-facing products.

ClickHouse is well-suited for this role because of its columnar compression, vectorized execution, and ability to scan billions of rows per second on commodity hardware.

## Designing Tables for Low Latency

The key to fast serving layer queries is designing the ORDER BY key to match your most common filter patterns:

```sql
CREATE TABLE user_metrics_serving (
    metric_date  Date,
    user_id      UInt64,
    app_id       UInt32,
    event_type   LowCardinality(String),
    count        UInt64,
    sum_value    Float64
) ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(metric_date)
ORDER BY (app_id, user_id, metric_date, event_type);
```

Queries filtering by `app_id` and `user_id` will read minimal data.

## Using Pre-Aggregated Materialized Views

Pre-aggregate at ingest time to avoid expensive GROUP BY on large tables at query time:

```sql
CREATE MATERIALIZED VIEW mv_daily_app_metrics
ENGINE = SummingMergeTree()
ORDER BY (metric_date, app_id, event_type)
AS
SELECT
    toDate(event_time) AS metric_date,
    app_id,
    event_type,
    count() AS count,
    sum(value) AS sum_value
FROM raw_events
GROUP BY metric_date, app_id, event_type;
```

Dashboard queries against this view return in milliseconds.

## Connection Pooling for the API Layer

ClickHouse recommends using persistent HTTP connections or a connection pool. In Node.js:

```bash
npm install @clickhouse/client
```

```text
import { createClient } from '@clickhouse/client';
const client = createClient({
  host: 'http://ch.internal:8123',
  database: 'analytics',
  username: 'api_reader',
  password: process.env.CH_PASSWORD,
  max_open_connections: 10,
});
```

## Caching API Responses

For queries that power public dashboards, cache the ClickHouse response at the API layer:

```text
GET /api/metrics/daily?app_id=42&days=30
  --> Check Redis cache (TTL 60s)
  --> On miss: query ClickHouse, store in Redis
  --> Return cached or fresh result
```

This reduces load on ClickHouse for repeated identical queries.

## Rate Limiting Queries

Prevent runaway queries from impacting the serving layer using query complexity limits:

```sql
SET max_rows_to_read = 1000000000;
SET max_execution_time = 5;
SET max_memory_usage = 4000000000;
```

Apply these as user-level defaults in `users.xml` for the API service account.

## Monitoring Serving Layer Latency

Track p50, p95, and p99 query latencies from the serving layer:

```sql
SELECT
    quantile(0.5)(query_duration_ms) AS p50_ms,
    quantile(0.95)(query_duration_ms) AS p95_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
  AND user = 'api_reader';
```

Alert via OneUptime when p95 exceeds your SLA threshold.

## Summary

A ClickHouse serving layer combines optimized table design, pre-aggregated materialized views, API-side caching, and query limits to deliver consistent sub-second latency for user-facing analytics products at any scale.
