# How to Design an Event Log Schema in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Event Log, Schema Design, Append-Only, Log Analytics, MergeTree

Description: Learn how to design an event log schema in ClickHouse for high-throughput append-only workloads like application logs, audit trails, and user events.

---

Event logs are one of the most common workloads in ClickHouse. They are append-only, high-volume, and queried primarily by time range with filtering on a few key dimensions. Getting the schema right delivers orders-of-magnitude performance improvements.

## Characteristics of Event Log Data

```text
- Append-only: records are never updated
- Time-ordered: most queries filter by timestamp range
- High cardinality: trace IDs, user IDs, request IDs are unique
- Semi-structured: extra fields vary per event type
- Read pattern: recent data more queried than old data
```

## Core Event Log Schema

```sql
CREATE TABLE event_logs (
    timestamp DateTime64(3),
    level LowCardinality(String),   -- DEBUG, INFO, WARN, ERROR
    service LowCardinality(String),
    host LowCardinality(String),
    trace_id String,
    span_id String,
    message String,
    attributes Map(String, String),
    -- Extracted common fields for fast filtering
    user_id UInt64 DEFAULT 0,
    request_id String DEFAULT '',
    status_code UInt16 DEFAULT 0
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (service, level, timestamp)
TTL toDateTime(timestamp) + INTERVAL 30 DAY
SETTINGS index_granularity = 8192;
```

## Application Log Schema

For structured application logs:

```sql
CREATE TABLE app_logs (
    ts DateTime64(6),
    severity LowCardinality(String),
    logger LowCardinality(String),
    message String,
    error_class LowCardinality(String),
    error_message String,
    stack_trace String,
    request_id String,
    user_id UInt64,
    environment LowCardinality(String),
    version LowCardinality(String),
    pod_name LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (environment, severity, ts)
TTL toDateTime(ts) + INTERVAL 14 DAY;
```

## Audit Log Schema

For compliance and security audit trails:

```sql
CREATE TABLE audit_logs (
    timestamp DateTime,
    actor_id UInt64,
    actor_type LowCardinality(String),  -- user, service, system
    action LowCardinality(String),
    resource_type LowCardinality(String),
    resource_id String,
    outcome LowCardinality(String),     -- success, failure, denied
    ip_address IPv4,
    user_agent String,
    metadata String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (actor_id, timestamp)
-- Audit logs: keep 7 years for compliance
TTL timestamp + INTERVAL 7 YEAR;
```

## Adding Full-Text Search Index

For log message searching:

```sql
ALTER TABLE event_logs
    ADD INDEX idx_message message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 4;

-- Efficient search
SELECT timestamp, service, message
FROM event_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
  AND message LIKE '%NullPointerException%'
LIMIT 100;
```

## Querying Common Log Patterns

Error rate over time:

```sql
SELECT
    toStartOfMinute(timestamp) AS minute,
    service,
    countIf(level = 'ERROR') AS errors,
    count() AS total,
    round(countIf(level = 'ERROR') / count() * 100, 2) AS error_pct
FROM event_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY minute, service
ORDER BY minute, service;
```

Top error messages:

```sql
SELECT
    message,
    count() AS occurrences,
    uniq(trace_id) AS unique_traces
FROM event_logs
WHERE timestamp >= now() - INTERVAL 24 HOUR
  AND level = 'ERROR'
GROUP BY message
ORDER BY occurrences DESC
LIMIT 20;
```

## Summary

A well-designed event log schema in ClickHouse uses LowCardinality for repeated string fields, daily or monthly partitioning for TTL efficiency, and an ORDER BY key that matches your primary filter pattern. Extract frequently filtered fields from semi-structured payloads into typed columns for fast predicate pushdown.
