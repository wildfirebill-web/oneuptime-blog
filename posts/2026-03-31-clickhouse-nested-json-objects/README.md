# How to Parse Nested JSON Objects in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Analytics, Query

Description: Learn how to parse nested JSON objects in ClickHouse using variadic path arguments with JSONExtract functions, with practical examples for multi-level object traversal.

---

ClickHouse's `JSONExtract*` family supports nested object traversal by accepting variadic path arguments after the JSON string. Each additional string argument descends one level into the object hierarchy. This eliminates the need to chain multiple function calls or extract intermediate sub-documents: `JSONExtractString(col, 'user', 'address', 'city')` navigates directly to `col.user.address.city` in a single call. Understanding this pattern is essential for working with real-world JSON payloads that have multiple levels of nesting.

## Variadic Path Syntax

```sql
-- Navigate two levels deep into a nested object
SELECT JSONExtractString(
    '{"user": {"name": "Alice", "address": {"city": "London", "zip": "EC1A"}}}',
    'user', 'name'
) AS user_name;
```

```text
user_name
Alice
```

## Three-Level Nesting

```sql
-- Extract a value three levels deep
SELECT JSONExtractString(
    '{"user": {"address": {"city": "London"}}}',
    'user', 'address', 'city'
) AS city;
```

```text
city
London
```

## Extracting Multiple Nested Fields

```sql
-- Pull several nested fields from an event payload column
SELECT
    event_id,
    JSONExtractString(payload, 'user', 'id')           AS user_id,
    JSONExtractString(payload, 'user', 'email')        AS user_email,
    JSONExtractString(payload, 'context', 'device')    AS device_type,
    JSONExtractString(payload, 'context', 'os')        AS os_name,
    JSONExtractInt(payload, 'metrics', 'duration_ms')  AS duration_ms
FROM events
LIMIT 10;
```

## Filtering on a Nested Field

```sql
-- Find requests from mobile devices based on a nested context field
SELECT
    event_id,
    event_time,
    JSONExtractString(payload, 'context', 'page', 'url') AS page_url
FROM events
WHERE JSONExtractString(payload, 'context', 'device') = 'mobile'
ORDER BY event_time DESC
LIMIT 20;
```

## Checking Nested Key Existence Before Extraction

```sql
-- Safely extract the nested field only when it exists
SELECT
    event_id,
    JSONHas(payload, 'geo', 'country')                            AS has_country,
    if(
        JSONHas(payload, 'geo', 'country') = 1,
        JSONExtractString(payload, 'geo', 'country'),
        'unknown'
    ) AS country
FROM events
LIMIT 10;
```

## Aggregating on Nested Values

```sql
-- Count errors per nested service and version
SELECT
    JSONExtractString(payload, 'service', 'name')    AS service_name,
    JSONExtractString(payload, 'service', 'version') AS service_version,
    count()                                          AS error_count
FROM error_logs
GROUP BY service_name, service_version
ORDER BY error_count DESC
LIMIT 20;
```

## Extracting a Nested Object as Raw JSON

When you want the entire nested sub-document rather than a single field, use `JSONExtractRaw`.

```sql
-- Extract the full nested address object as a JSON string
SELECT
    user_id,
    JSONExtractRaw(profile, 'address') AS raw_address_json
FROM users
WHERE JSONHas(profile, 'address') = 1
LIMIT 10;
```

## Materialized Columns for Frequently Accessed Nested Fields

```sql
-- Materialize a frequently filtered nested field to avoid repeated parsing
ALTER TABLE events
    ADD COLUMN country String
    MATERIALIZED JSONExtractString(payload, 'geo', 'country');
```

## Summary

ClickHouse supports nested JSON object traversal through variadic path arguments on all `JSONExtract*` functions. Each additional string argument steps one level deeper into the object. Always combine deep extraction with `JSONHas` using the same path when the field may be absent. For fields that appear in `WHERE` clauses or `GROUP BY` at high frequency, materialize them at insert time to eliminate repeated parsing cost.
