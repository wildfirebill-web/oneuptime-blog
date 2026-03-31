# How to Use Comparison Operators in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Comparison, Operator, SQL, Filter

Description: Explore how comparison operators work in ClickHouse, including equality, range checks, and behavior with Nullable types and strings.

---

## Standard Comparison Operators

ClickHouse supports the full set of SQL comparison operators: `=`, `!=` (or `<>`), `<`, `>`, `<=`, `>=`. These return a `UInt8` value (1 for true, 0 for false) rather than a boolean type.

```sql
SELECT
  5 = 5 AS eq,         -- 1
  5 != 3 AS neq,       -- 1
  5 < 10 AS lt,        -- 1
  5 > 10 AS gt,        -- 0
  5 <= 5 AS lte,       -- 1
  5 >= 6 AS gte        -- 0
```

## Using Comparisons in WHERE Clauses

```sql
SELECT user_id, revenue
FROM orders
WHERE
  revenue > 100
  AND status = 'completed'
  AND order_date >= '2025-01-01'
```

ClickHouse can use primary key and sort key indexes for range comparisons on the leading columns of `ORDER BY`.

## String Comparisons

String comparisons use lexicographic ordering based on byte values. Case sensitivity is the default.

```sql
SELECT *
FROM products
WHERE category = 'Electronics'
  AND name < 'M'  -- all names before 'M' alphabetically
```

For case-insensitive comparison, use `ilike` or `lower()`.

```sql
SELECT * FROM products WHERE lower(name) = 'laptop'
```

## Comparing with Nullable Columns

Comparing a `Nullable` column to a value returns `NULL` rather than `0` when the column is `NULL`. This can cause rows to silently disappear from filter results.

```sql
-- This may miss rows where score IS NULL
SELECT * FROM users WHERE score != 0;

-- Explicit NULL handling
SELECT * FROM users WHERE score != 0 OR score IS NULL;
```

Use `isNull` and `isNotNull` functions for clarity, or `assumeNotNull` when you know the column has no nulls.

## Numeric Type Comparisons

ClickHouse promotes types during comparison. Comparing a `UInt64` column to a negative literal always returns false because `UInt64` cannot represent negative values.

```sql
-- This always returns 0 rows - UInt64 cannot be < 0
SELECT * FROM events WHERE user_id < -1;

-- Use the correct type
SELECT * FROM events WHERE user_id < toUInt64(1000)
```

## Tuple Comparison

ClickHouse supports tuple comparison, which is useful for composite key range queries.

```sql
SELECT *
FROM events
WHERE (user_id, event_time) > (1000, '2025-06-01 00:00:00')
```

## Summary

ClickHouse comparison operators return `UInt8` rather than a boolean, behave as expected on numeric and string types, and require explicit NULL checks when working with `Nullable` columns. Use tuple comparisons for composite key range scans and be mindful of unsigned integer types when comparing against negative literals.
