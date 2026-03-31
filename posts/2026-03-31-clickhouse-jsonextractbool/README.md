# How to Use JSONExtractBool() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Analytics, Query

Description: Learn how JSONExtractBool() extracts boolean values from JSON strings in ClickHouse, returning a UInt8 value of 1 or 0 for use in filters and aggregations.

---

`JSONExtractBool` reads a JSON boolean field (`true` or `false`) from a JSON string and returns a `UInt8`: `1` for `true` and `0` for `false`. It returns `0` when the key is absent or the value is not a JSON boolean. This makes it useful for toggling logic, feature-flag analysis, and conditional aggregations where flags are stored inside JSON payloads.

## Basic Usage

```sql
-- Extract a boolean from a JSON literal
SELECT
    JSONExtractBool('{"active": true, "deleted": false}', 'active')   AS active,
    JSONExtractBool('{"active": true, "deleted": false}', 'deleted')  AS deleted;
```

```text
active  deleted
1       0
```

The return type is `UInt8`, so it integrates directly with arithmetic and conditional expressions.

## Filtering Rows by a Boolean Flag

```sql
-- Find events where the JSON payload marks the request as internal
SELECT
    event_id,
    event_time,
    JSONExtractString(payload, 'endpoint') AS endpoint
FROM events
WHERE JSONExtractBool(payload, 'is_internal') = 1
ORDER BY event_time DESC
LIMIT 20;
```

## Counting True vs False

Because `UInt8` is numeric, `sum` and `countIf` work directly on the extracted value.

```sql
-- Tally enabled vs disabled feature flags per experiment
SELECT
    JSONExtractString(payload, 'experiment_id')         AS experiment_id,
    sum(JSONExtractBool(payload, 'flag_enabled'))       AS enabled_count,
    countIf(JSONExtractBool(payload, 'flag_enabled') = 0) AS disabled_count
FROM feature_events
GROUP BY experiment_id
ORDER BY experiment_id;
```

## Nested Boolean Fields

Pass additional string arguments to navigate nested objects.

```sql
-- Extract {"settings": {"notifications": {"email": true}}}
SELECT
    user_id,
    JSONExtractBool(preferences, 'settings', 'notifications', 'email') AS email_notifications
FROM users
LIMIT 10;
```

## Distinguishing Missing from False

`JSONExtractBool` returns `0` for both `false` and a missing key. Use `JSONHas` to tell them apart.

```sql
SELECT
    event_id,
    JSONHas(payload, 'verified')              AS has_verified_field,
    JSONExtractBool(payload, 'verified')      AS verified
FROM events
WHERE JSONHas(payload, 'verified') = 1
LIMIT 10;
```

## Using in a CASE Expression

```sql
-- Label rows based on the boolean flag value
SELECT
    event_id,
    CASE
        WHEN JSONExtractBool(payload, 'is_retry') = 1 THEN 'retry'
        ELSE 'original'
    END AS request_type
FROM requests
LIMIT 10;
```

## Materialized Column for Repeated Filtering

If `is_active` is filtered on every query, materialize it to avoid repeated JSON parsing.

```sql
ALTER TABLE users
    ADD COLUMN is_active UInt8
    MATERIALIZED JSONExtractBool(profile_json, 'is_active');
```

Queries can then filter on `is_active` directly and use index support.

## Summary

`JSONExtractBool` returns `1` or `0` from a JSON boolean field and integrates cleanly with arithmetic, `countIf`, and `CASE` expressions. Remember that a missing key and an explicit `false` both return `0`, so combine it with `JSONHas` when the distinction matters. For columns that are filtered frequently, a materialized column eliminates repeated parsing cost.
