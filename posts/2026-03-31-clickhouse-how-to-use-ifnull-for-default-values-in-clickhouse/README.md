# How to Use ifNull() for Default Values in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Null Handling, ifNull, Default Values, Sql

Description: Learn how to use ifNull() in ClickHouse to substitute default values for NULL entries in Nullable columns, simplifying data transformation and reporting.

---

## Overview

`ifNull(value, default)` returns the first argument if it is not NULL, otherwise returns the second argument. It is ClickHouse's equivalent of SQL `COALESCE(value, default)` for two arguments, and is the primary tool for replacing NULLs with meaningful defaults.

## Basic Usage

```sql
SELECT
    ifNull(NULL, 'fallback')           AS str_default,
    ifNull('hello', 'fallback')        AS str_value,
    ifNull(NULL, 0)                    AS num_default,
    ifNull(42, 0)                      AS num_value
```

Output:

```text
str_default | str_value | num_default | num_value
fallback    | hello     | 0           | 42
```

## With Nullable Column Values

```sql
CREATE TABLE orders
(
    order_id   UInt64,
    customer   String,
    coupon     Nullable(String),
    discount   Nullable(Float64)
)
ENGINE = MergeTree()
ORDER BY order_id;

INSERT INTO orders VALUES
(1, 'Alice', 'SAVE10', 10.0),
(2, 'Bob',   NULL,     NULL),
(3, 'Carol', 'VIP20',  20.0);
```

Apply defaults for NULL fields:

```sql
SELECT
    order_id,
    customer,
    ifNull(coupon,   'NO_COUPON') AS coupon_code,
    ifNull(discount, 0.0)        AS discount_pct
FROM orders
```

Output:

```text
order_id | customer | coupon_code | discount_pct
1        | Alice    | SAVE10      | 10.0
2        | Bob      | NO_COUPON   | 0.0
3        | Carol    | VIP20       | 20.0
```

## Using ifNull in Aggregations

NULL values are excluded from `sum()` and `avg()` by default. Use `ifNull` before aggregating to treat NULLs as zero:

```sql
SELECT
    sum(ifNull(discount, 0.0))        AS total_discount,
    avg(ifNull(discount, 0.0))        AS avg_discount_incl_nulls,
    avg(discount)                      AS avg_discount_excl_nulls
FROM orders
```

## ifNull vs COALESCE

For two arguments, `ifNull` and `COALESCE` are equivalent:

```sql
SELECT ifNull(coupon, 'NONE') AS using_ifNull FROM orders
SELECT COALESCE(coupon, 'NONE') AS using_coalesce FROM orders
-- Identical results
```

For more than two fallback candidates, use `COALESCE`:

```sql
SELECT COALESCE(preferred_name, username, email, 'Anonymous') AS display_name
FROM user_profiles
```

## Chaining for Multi-Level Fallback

Use nested `ifNull` calls for multi-step fallbacks:

```sql
SELECT
    ifNull(
        preferred_email,
        ifNull(work_email, personal_email)
    ) AS best_email
FROM contacts
```

## Using ifNull with Date and DateTime

```sql
SELECT
    order_id,
    ifNull(shipped_at, toDateTime('1970-01-01 00:00:00')) AS shipped_at_or_epoch,
    isNull(shipped_at) AS is_pending
FROM orders
```

## Dashboard-Ready Queries

Replace NULLs in metrics columns for clean dashboard output:

```sql
SELECT
    toDate(event_time)                       AS date,
    ifNull(sum(revenue), 0.0)               AS total_revenue,
    ifNull(count(DISTINCT user_id), 0)      AS unique_users
FROM daily_stats
WHERE date >= today() - 30
GROUP BY date
ORDER BY date
```

## ifNull vs if(isNull(...), default, value)

These are equivalent; `ifNull` is more concise:

```sql
-- Verbose
SELECT if(isNull(coupon), 'NONE', coupon) AS result FROM orders

-- Concise
SELECT ifNull(coupon, 'NONE') AS result FROM orders
```

## Summary

`ifNull(value, default)` is the standard ClickHouse function for replacing NULL with a default value. It is equivalent to `COALESCE` for two arguments and works with any Nullable type. Use it in SELECT lists to normalize output, in aggregations to treat NULLs as zero, and in nested form for multi-level fallback chains.
