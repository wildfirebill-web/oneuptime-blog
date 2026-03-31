# How to Store and Query Distributed Traces in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed Tracing, OpenTelemetry, Observability, Span

Description: Store and query distributed traces in ClickHouse using an OpenTelemetry-compatible schema and efficient span retrieval queries.

---

Distributed tracing gives you end-to-end visibility across microservices. ClickHouse is increasingly used as the storage backend for trace data because it handles high write throughput and fast span lookups better than many purpose-built tracing stores.

## Traces Table Schema

```sql
CREATE TABLE otel_traces
(
    trace_id FixedString(32),
    span_id FixedString(16),
    parent_span_id FixedString(16) DEFAULT '',
    trace_state String DEFAULT '',
    span_name LowCardinality(String),
    span_kind UInt8,
    service_name LowCardinality(String),
    resource_attributes Map(String, String),
    span_attributes Map(String, String),
    start_time_unix_nano UInt64,
    end_time_unix_nano UInt64,
    duration_nano UInt64,
    status_code UInt8 DEFAULT 0,
    status_message String DEFAULT '',
    events Nested(
        event_time_unix_nano UInt64,
        event_name String,
        event_attributes Map(String, String)
    )
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(toDateTime(intDiv(start_time_unix_nano, 1000000000)))
ORDER BY (service_name, toDateTime(intDiv(start_time_unix_nano, 1000000000)), trace_id);
```

## Retrieve All Spans for a Trace

```sql
SELECT
    span_id,
    parent_span_id,
    service_name,
    span_name,
    toDateTime(intDiv(start_time_unix_nano, 1000000000)) AS start_time,
    round(duration_nano / 1e6, 2) AS duration_ms,
    status_code
FROM otel_traces
WHERE trace_id = '4bf92f3577b34da6a3ce929d0e0e4736'
ORDER BY start_time_unix_nano ASC;
```

## Find Root Spans (Entry Points)

```sql
SELECT
    trace_id,
    span_id,
    service_name,
    span_name,
    toDateTime(intDiv(start_time_unix_nano, 1000000000)) AS start_time,
    round(duration_nano / 1e6, 2) AS duration_ms
FROM otel_traces
WHERE parent_span_id = ''
  AND toDateTime(intDiv(start_time_unix_nano, 1000000000)) >= now() - INTERVAL 1 HOUR
  AND status_code = 2
ORDER BY duration_ms DESC
LIMIT 20;
```

## Slow Spans by Operation

```sql
SELECT
    service_name,
    span_name,
    count() AS span_count,
    round(quantile(0.95)(duration_nano) / 1e6, 2) AS p95_ms,
    round(quantile(0.99)(duration_nano) / 1e6, 2) AS p99_ms
FROM otel_traces
WHERE toDateTime(intDiv(start_time_unix_nano, 1000000000)) >= now() - INTERVAL 1 HOUR
GROUP BY service_name, span_name
ORDER BY p99_ms DESC
LIMIT 15;
```

## Trace Sampling - Find Traces with High Latency

```sql
SELECT DISTINCT trace_id
FROM otel_traces
WHERE parent_span_id = ''
  AND duration_nano > 5000000000
  AND toDateTime(intDiv(start_time_unix_nano, 1000000000)) >= now() - INTERVAL 30 MINUTE
LIMIT 50;
```

## Span Error Rate by Service

```sql
SELECT
    service_name,
    count() AS total_spans,
    countIf(status_code = 2) AS error_spans,
    round(countIf(status_code = 2) * 100.0 / count(), 3) AS error_rate_pct
FROM otel_traces
WHERE toDateTime(intDiv(start_time_unix_nano, 1000000000)) >= now() - INTERVAL 1 HOUR
GROUP BY service_name
ORDER BY error_rate_pct DESC;
```

## Query Span Events for Exceptions

```sql
SELECT
    trace_id,
    span_id,
    service_name,
    events.event_name,
    events.event_attributes
FROM otel_traces
ARRAY JOIN events
WHERE events.event_name = 'exception'
  AND toDateTime(intDiv(start_time_unix_nano, 1000000000)) >= now() - INTERVAL 1 HOUR
LIMIT 20;
```

## Summary

ClickHouse stores distributed traces efficiently using a columnar schema with nanosecond timestamps and `Map` types for flexible attributes. Root span queries, latency percentiles, error rates, and span event lookups all run in milliseconds on large datasets. The combination of daily partitioning and ordering by service name plus timestamp ensures that trace retrieval by service or time range stays fast.
