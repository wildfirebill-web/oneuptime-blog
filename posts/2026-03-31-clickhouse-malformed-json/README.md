# How to Handle Malformed JSON in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Data Quality, Analytics, Query

Description: Learn how ClickHouse handles malformed JSON in JSONExtract functions, how to detect invalid payloads using isValidJSON(), and strategies for quarantining bad data.

---

Real-world JSON pipelines inevitably encounter malformed payloads: truncated strings, missing closing braces, stray control characters, or empty fields. ClickHouse's `JSONExtract*` functions do not raise errors on malformed input; they silently return type defaults (`0`, `''`, empty array). The `isValidJSON` function lets you explicitly test whether a string is valid JSON, enabling quarantine pipelines and data quality checks. This guide covers detecting, filtering, and routing malformed JSON in ClickHouse.

## Silent Default Behavior

```sql
-- Malformed JSON returns defaults rather than errors
SELECT
    JSONExtractInt('{"id": 1, "broken":', 'id')   AS id_from_broken,
    JSONExtractString('not json at all', 'name')  AS name_from_garbage;
```

```text
id_from_broken  name_from_garbage
0               (empty string)
```

This means you cannot rely on zero or empty string alone to detect bad payloads.

## Detecting Invalid JSON with isValidJSON

```sql
-- Check whether payloads are valid JSON
SELECT
    isValidJSON('{"key": "value"}')  AS valid,
    isValidJSON('{"key": }')         AS invalid,
    isValidJSON('')                  AS empty_string,
    isValidJSON('null')              AS json_null;
```

```text
valid  invalid  empty_string  json_null
1      0        0             1
```

Note: the JSON literal `null` is valid JSON and returns `1`.

## Auditing a Column for Invalid Rows

```sql
-- Count valid vs invalid payloads in an ingestion table
SELECT
    isValidJSON(payload) AS is_valid,
    count()              AS row_count
FROM events_raw
GROUP BY is_valid;
```

## Filtering Out Bad Rows Before Processing

```sql
-- Only process valid JSON payloads
SELECT
    event_id,
    JSONExtractString(payload, 'user_id') AS user_id,
    JSONExtractString(payload, 'type')    AS event_type
FROM events_raw
WHERE isValidJSON(payload) = 1
LIMIT 20;
```

## Quarantining Malformed Rows

Use a materialized view to route invalid rows to a separate table for investigation.

```sql
-- Table for bad payloads
CREATE TABLE events_malformed (
    received_at DateTime,
    raw_payload String,
    source      String
) ENGINE = MergeTree()
ORDER BY received_at;

-- Materialized view routing invalid rows
CREATE MATERIALIZED VIEW events_malformed_mv TO events_malformed AS
SELECT
    received_at,
    payload   AS raw_payload,
    source
FROM events_raw
WHERE isValidJSON(payload) = 0;
```

## Checking for Empty or Null Payloads

```sql
-- Identify rows with missing, empty, or invalid payloads
SELECT
    event_id,
    length(payload)         AS payload_length,
    isValidJSON(payload)    AS valid_json
FROM events_raw
WHERE
    payload = ''
    OR payload IS NULL
    OR isValidJSON(payload) = 0
ORDER BY received_at DESC
LIMIT 20;
```

## Providing Fallbacks for Specific Fields

When you suspect a field may be absent due to truncation rather than a completely invalid document, combine `isValidJSON` with `JSONHas`.

```sql
-- Use fallback values for potentially malformed records
SELECT
    event_id,
    if(
        isValidJSON(payload) = 1 AND JSONHas(payload, 'user_id'),
        JSONExtractString(payload, 'user_id'),
        'unknown'
    ) AS user_id
FROM events_raw
LIMIT 20;
```

## Reporting on Malformed JSON Over Time

```sql
-- Daily count of malformed payloads for a data quality dashboard
SELECT
    toDate(received_at)     AS day,
    countIf(isValidJSON(payload) = 0) AS malformed_count,
    count()                           AS total_count,
    round(
        100.0 * countIf(isValidJSON(payload) = 0) / count(),
        2
    )                                 AS malformed_pct
FROM events_raw
WHERE received_at >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day;
```

## Summary

ClickHouse `JSONExtract*` functions return silent defaults on malformed input, so use `isValidJSON` to explicitly detect bad payloads. Build quarantine pipelines with materialized views to route invalid rows to a separate table before they pollute downstream aggregations. Combine `isValidJSON` with `JSONHas` for granular field-level safety checks when full JSON validity is confirmed but individual fields may still be absent.
