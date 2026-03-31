# How to Use nullIf() to Convert Values to NULL in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL Handling, NULLIF, SQL, Data Cleaning

Description: Learn how to use nullIf() in ClickHouse to convert specific sentinel values to NULL, enabling cleaner aggregations and conditional NULL handling.

---

## Overview

`nullIf(value, sentinel)` returns NULL if `value` equals `sentinel`, otherwise returns `value`. It is the inverse of `ifNull()` - instead of replacing NULLs with defaults, it converts specific placeholder values (like empty strings or -1) into NULL so ClickHouse NULL semantics apply.

## Basic Usage

```sql
SELECT
    nullIf(0, 0)       AS zero_becomes_null,
    nullIf(5, 0)       AS five_stays,
    nullIf('', '')     AS empty_becomes_null,
    nullIf('hello', '') AS hello_stays
```

Output:

```text
zero_becomes_null | five_stays | empty_becomes_null | hello_stays
NULL              | 5          | NULL               | hello
```

## The Return Type is Nullable

```sql
SELECT toTypeName(nullIf(42, 0)) AS type
-- Nullable(UInt8)
```

`nullIf` always returns a `Nullable(T)` type regardless of whether the sentinel matches.

## Cleaning Empty String Sentinels

Databases often store empty strings instead of NULL for optional fields. Use `nullIf` to normalize them:

```sql
SELECT
    user_id,
    nullIf(phone_number, '')       AS phone,
    nullIf(address_line2, '')      AS address_line2,
    nullIf(referral_code, 'NONE')  AS referral
FROM user_registrations
```

## Preventing Division by Zero

A classic use case: convert a zero denominator to NULL to avoid division-by-zero errors (NULL propagates through arithmetic):

```sql
SELECT
    team_id,
    total_sales,
    total_calls,
    total_sales / nullIf(total_calls, 0) AS sales_per_call
FROM team_stats
```

If `total_calls` is 0, `sales_per_call` will be NULL instead of causing an exception.

## Excluding Sentinel Values from Aggregations

NULLs are excluded from `avg()`, `sum()`, and `count()`. Use `nullIf` to exclude sentinel values from aggregations:

```sql
SELECT
    avg(nullIf(response_time_ms, -1)) AS avg_response_time,
    min(nullIf(response_time_ms, -1)) AS min_response_time
FROM api_logs
-- -1 is used as "timeout" sentinel; exclude from timing stats
```

## Combining with ifNull

Use `nullIf` to convert sentinels to NULL, then `ifNull` to apply a different default:

```sql
SELECT
    ifNull(nullIf(status_code, 0), -1) AS sanitized_status
FROM api_responses
-- 0 becomes NULL, then NULL becomes -1
-- Any non-zero value stays as-is
```

## Practical Example - Removing Test Users

Mark test user data as NULL for exclusion from analytics:

```sql
SELECT
    date,
    count(nullIf(user_id, 0))     AS real_user_events,
    sum(nullIf(revenue, 0.0))     AS real_revenue
FROM events
GROUP BY date
ORDER BY date
```

## Detecting Sentinel vs. Real Zeros

Use `isNull(nullIf(val, 0))` to detect actual zeros vs. other values in a Boolean expression:

```sql
SELECT
    metric_value,
    isNull(nullIf(metric_value, 0)) AS is_zero
FROM metrics
```

## nullIf vs. CASE

`nullIf(a, b)` is a concise equivalent of:

```sql
CASE WHEN a = b THEN NULL ELSE a END
```

The function form is preferred for readability in simple cases.

## Summary

`nullIf(value, sentinel)` converts a specific sentinel value to NULL, enabling ClickHouse NULL semantics to apply for aggregations, division safety, and conditional logic. It pairs naturally with `ifNull()` for full null-substitution pipelines and is the preferred way to clean up empty strings, -1 placeholders, and other sentinel conventions in ingested data.
