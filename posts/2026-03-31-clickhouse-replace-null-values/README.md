# How to Replace NULL Values in ClickHouse Query Results

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL, Query, Data Cleaning, coalesce, ifNull

Description: Learn how to replace NULL values in ClickHouse query results using ifNull, coalesce, and other built-in functions for cleaner analytics.

---

## Why NULL Replacement Matters

NULL values in analytical datasets can silently distort aggregations, break comparisons, and confuse downstream consumers. ClickHouse provides several idiomatic functions to replace NULLs at query time without modifying the underlying data.

## Using ifNull

`ifNull(x, alt)` returns `alt` when `x` is NULL, otherwise returns `x`.

```sql
SELECT
    user_id,
    ifNull(country, 'Unknown') AS country,
    ifNull(revenue, 0.0) AS revenue
FROM user_events
LIMIT 10;
```

This is the most common pattern for replacing NULLs with a default value.

## Using coalesce

`coalesce` accepts multiple arguments and returns the first non-NULL value.

```sql
SELECT
    user_id,
    coalesce(preferred_name, full_name, email, 'Anonymous') AS display_name
FROM users
LIMIT 20;
```

Use `coalesce` when you have a priority chain of fallback columns.

## Using nullIf to Create NULLs Conditionally

Sometimes you want to do the reverse - turn a sentinel value into NULL first, then replace it:

```sql
SELECT
    product_id,
    ifNull(nullIf(price, 0), 9.99) AS safe_price
FROM products;
```

`nullIf(price, 0)` converts zeros to NULL, and then `ifNull` replaces them with 9.99.

## Replacing NULLs in Arrays

For array columns, use `arrayMap` combined with `ifNull`:

```sql
SELECT
    arrayMap(x -> ifNull(x, 0), scores) AS filled_scores
FROM student_results;
```

## Replacing NULLs in Aggregations

By default, aggregate functions like `sum` and `avg` ignore NULLs. But when you need explicit control:

```sql
SELECT
    department,
    avg(ifNull(salary, 0)) AS avg_salary_with_zeros,
    avgIf(salary, salary IS NOT NULL) AS avg_salary_excluding_nulls
FROM employees
GROUP BY department;
```

## Using COALESCE in JOINs

Left joins often produce NULLs for unmatched rows. Fill them with defaults:

```sql
SELECT
    o.order_id,
    coalesce(c.name, 'Guest') AS customer_name,
    coalesce(c.tier, 'standard') AS tier
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

## Materialized Columns as a Permanent Solution

If you repeatedly replace NULLs for the same column, consider a materialized column:

```sql
ALTER TABLE user_events
    ADD COLUMN country_filled String
    MATERIALIZED ifNull(country, 'Unknown');
```

This computes the replacement once at insert time, avoiding repeated function calls at query time.

## Summary

ClickHouse offers `ifNull`, `coalesce`, and `nullIf` as the primary tools for NULL replacement. Use `ifNull` for simple single-column defaults, `coalesce` for priority fallback chains, and materialized columns when the replacement is always the same and query performance is critical.
