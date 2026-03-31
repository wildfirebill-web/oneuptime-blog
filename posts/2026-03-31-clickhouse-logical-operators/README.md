# How to Use Logical Operators (AND, OR, NOT) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Logical Operator, AND, OR, NOT, SQL

Description: Learn how AND, OR, and NOT logical operators work in ClickHouse, including short-circuit evaluation, precedence rules, and Nullable handling.

---

## AND, OR, and NOT Basics

ClickHouse supports `AND`, `OR`, and `NOT` as logical operators returning `UInt8` (1 or 0). You can also use `&&`, `||`, and `!` as symbolic equivalents.

```sql
SELECT
  1 AND 1 AS both_true,   -- 1
  1 AND 0 AS one_false,   -- 0
  0 OR 1 AS one_true,     -- 1
  NOT 1 AS not_true,      -- 0
  NOT 0 AS not_false      -- 1
```

## Operator Precedence

`NOT` has higher precedence than `AND`, which has higher precedence than `OR`. Use parentheses to make complex conditions explicit.

```sql
-- Without parens: interpreted as (status = 'active' AND type = 'premium') OR type = 'trial'
SELECT * FROM users
WHERE status = 'active' AND type = 'premium' OR type = 'trial';

-- With parens: status = 'active' AND (type = 'premium' OR type = 'trial')
SELECT * FROM users
WHERE status = 'active' AND (type = 'premium' OR type = 'trial');
```

## Combining Logical Operators in Filters

```sql
SELECT
  user_id,
  event_type,
  event_time
FROM events
WHERE
  event_date >= '2025-01-01'
  AND event_date < '2026-01-01'
  AND (event_type = 'purchase' OR event_type = 'refund')
  AND NOT is_bot
```

## Short-Circuit Evaluation

ClickHouse evaluates `AND` and `OR` expressions left to right and may short-circuit - if the first operand of `AND` is false, the second is not evaluated, and vice versa for `OR`. This matters when the right-hand side is expensive.

```sql
-- Place the cheap, high-selectivity check first
SELECT * FROM logs
WHERE level = 'ERROR'
  AND extractURLParameter(url, 'trace_id') != ''
```

## Logical Operators with Nullable Values

When any operand is `NULL`, the result follows three-valued logic. `NULL AND 1` returns `NULL`, not `0`.

```sql
SELECT
  NULL AND 1,   -- NULL
  NULL OR 1,    -- 1
  NULL OR 0,    -- NULL
  NOT NULL      -- NULL
```

To safely filter on nullable columns, combine with `isNull` or `coalesce`.

```sql
SELECT * FROM users
WHERE coalesce(is_verified, 0) = 1
  AND (region = 'US' OR region IS NULL)
```

## Using AND/OR in HAVING

```sql
SELECT
  event_type,
  count() AS total,
  uniq(user_id) AS users
FROM events
GROUP BY event_type
HAVING total > 1000 AND users > 100
```

## Summary

ClickHouse logical operators `AND`, `OR`, and `NOT` work on `UInt8` values with standard SQL precedence rules. Always use parentheses in complex conditions to avoid precedence surprises. Be aware of three-valued logic when `Nullable` columns are involved - use `coalesce` or explicit `IS NULL` checks to get deterministic results.
