# How to Use groupConcat() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Aggregate Function, groupConcat, String Aggregation, SQL

Description: Learn how to use groupConcat() in ClickHouse to aggregate multiple string values within a group into a single delimited string, with separator and limit control.

---

`groupConcat()` is a string aggregation function that concatenates all values in a group into a single string, separated by a configurable delimiter. It is the ClickHouse equivalent of `GROUP_CONCAT()` in MySQL or `string_agg()` in PostgreSQL, and was added as a more ergonomic alternative to `arrayStringConcat(groupArray(...), sep)`.

## Function Signatures

```text
groupConcat(sep)(expr)          -- aggregate expr values, join with sep
groupConcat(sep, limit)(expr)   -- same, but collect at most limit values
groupConcat(expr)               -- aggregate with default separator (no separator)
```

The separator is passed as a parameter to the aggregate function combinator, not as a column argument. This makes it composable with `-If`, `-Array`, and `-State` combinators.

## Basic Usage

```sql
SELECT
    department,
    groupConcat(', ')(employee_name) AS employees
FROM employees
GROUP BY department
ORDER BY department;
```

```text
┌─department──┬─employees──────────────────────┐
│ Engineering │ Alice, Bob, Carol, Dave         │
│ Marketing   │ Eve, Frank                      │
│ Sales       │ Grace, Heidi, Ivan              │
└─────────────┴────────────────────────────────┘
```

## Equivalent Legacy Pattern

Before `groupConcat()` was available, the same result required nesting two functions.

```sql
-- Old pattern
SELECT department, arrayStringConcat(groupArray(employee_name), ', ') AS employees
FROM employees GROUP BY department;

-- New pattern (equivalent)
SELECT department, groupConcat(', ')(employee_name) AS employees
FROM employees GROUP BY department;
```

## Limiting the Number of Values

The optional second parameter caps how many values are collected before concatenation. This is useful to prevent unbounded string growth on high-cardinality groups.

```sql
SELECT
    category,
    groupConcat(', ', 5)(product_name) AS sample_products
FROM products
GROUP BY category;
```

## Preserving Order

`groupConcat` does not guarantee any particular order across rows. To control order, pre-sort with a subquery or use `arrayStringConcat(arraySort(groupArray(...)), sep)`.

```sql
SELECT
    user_id,
    arrayStringConcat(
        arraySort(groupArray(tag)),
        ', '
    ) AS sorted_tags
FROM user_tags
GROUP BY user_id;
```

## Combining with -If Combinator

`groupConcatIf` collects only values where a condition is true.

```sql
SELECT
    order_id,
    groupConcatIf(', ')(item_name, quantity > 1) AS multi_quantity_items
FROM order_items
GROUP BY order_id;
```

## Building Comma-Separated Tag Lists

A common use case is generating a human-readable list of tags for each row to use in reports or exports.

```sql
SELECT
    article_id,
    title,
    groupConcat(', ')(tag) AS tags
FROM articles
JOIN article_tags USING (article_id)
GROUP BY article_id, title
ORDER BY article_id;
```

## Generating SQL IN Lists for Debugging

During development, `groupConcat` is handy for building IN-list strings for inspection.

```sql
SELECT groupConcat(', ')(toString(user_id)) AS user_id_list
FROM (
    SELECT DISTINCT user_id
    FROM failed_payments
    WHERE event_time >= now() - INTERVAL 1 HOUR
);
```

## Differences from arrayStringConcat + groupArray

| Feature | `groupConcat` | `arrayStringConcat(groupArray(...))` |
|---|---|---|
| Memory | Same (both build array internally) | Same |
| Separator location | Function parameter | Second argument to arrayStringConcat |
| Combinator support | Native (`-If`, `-State`, etc.) | Wrapping required |
| Readability | Cleaner one-liner | More explicit two-step |

## Summary

`groupConcat()` is the clean, combinator-friendly way to aggregate string values within a group into a single delimited string. It replaces the common `arrayStringConcat(groupArray(...), sep)` idiom with a single aggregate function that integrates natively with ClickHouse combinators like `-If` and `-State`. Use the optional limit parameter to cap group sizes and prevent runaway memory use in high-cardinality aggregations.
