# How to Use Materialized Views for Data Transformation in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Data Transformation, ETL, Pipeline

Description: Learn how to use ClickHouse materialized views to transform raw data on ingest, normalizing formats, extracting fields, and reshaping records automatically.

---

## Transformation vs. Aggregation

While materialized views are often used for aggregation, they are equally powerful for row-level transformations - extracting JSON fields, converting data types, normalizing strings, and filtering out unwanted records.

## Parsing JSON on Ingest

Raw events often arrive as JSON strings. Use a materialized view to extract fields:

```sql
CREATE TABLE raw_log_events (
    event_time DateTime,
    raw_data String
) ENGINE = MergeTree()
ORDER BY event_time;

CREATE TABLE parsed_log_events (
    event_time DateTime,
    user_id UInt64,
    action LowCardinality(String),
    resource String,
    ip_address String,
    duration_ms UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);

CREATE MATERIALIZED VIEW mv_parse_log_events
TO parsed_log_events
AS SELECT
    event_time,
    JSONExtractUInt(raw_data, 'user_id') AS user_id,
    JSONExtractString(raw_data, 'action') AS action,
    JSONExtractString(raw_data, 'resource') AS resource,
    JSONExtractString(raw_data, 'ip') AS ip_address,
    JSONExtractUInt(raw_data, 'duration_ms') AS duration_ms
FROM raw_log_events;
```

## Normalizing and Cleaning Data

Transform messy input data into clean, consistent formats:

```sql
CREATE MATERIALIZED VIEW mv_clean_events
TO clean_events
AS SELECT
    toDateTime(timestamp_str) AS event_time,
    lower(trim(user_email)) AS user_email,
    upper(country_code) AS country_code,
    if(amount < 0, 0, amount) AS amount,
    replaceAll(phone_number, '-', '') AS phone_number
FROM raw_events;
```

## Filtering on Ingest

Only keep relevant records in the transformed table:

```sql
CREATE MATERIALIZED VIEW mv_error_events_only
TO error_events
AS SELECT
    event_time,
    service,
    message,
    stack_trace
FROM application_logs
WHERE level = 'ERROR'
  AND service != 'health-check';
```

## Splitting a Stream into Multiple Tables

Route different event types to different tables:

```sql
-- Table for purchase events
CREATE MATERIALIZED VIEW mv_purchase_events
TO purchase_events
AS SELECT event_time, user_id, amount, product_id
FROM raw_events
WHERE event_type = 'purchase';

-- Table for page view events
CREATE MATERIALIZED VIEW mv_pageview_events
TO pageview_events
AS SELECT event_time, user_id, page_url, referrer
FROM raw_events
WHERE event_type = 'pageview';
```

## Enriching with Dictionary Lookups

Add context from a dictionary during transformation:

```sql
CREATE MATERIALIZED VIEW mv_enriched_events
TO enriched_events
AS SELECT
    event_time,
    user_id,
    ip_address,
    dictGetString('geo_dict', 'country', tuple(IPv4StringToNum(ip_address))) AS country,
    dictGetString('geo_dict', 'city', tuple(IPv4StringToNum(ip_address))) AS city
FROM raw_events;
```

## Summary

ClickHouse materialized views are not just for aggregations - they are powerful ETL tools. Use them to parse JSON, normalize strings, filter records, route events to multiple tables, and enrich data with dictionary lookups, all automatically at insert time without extra infrastructure.
