# How to Use JSON_ARRAYAGG() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Aggregate Function

Description: Learn how to use MySQL's JSON_ARRAYAGG() aggregate function to collect multiple rows into a single JSON array with ordering and grouping.

---

## What Is JSON_ARRAYAGG()?

`JSON_ARRAYAGG()` is an aggregate function introduced in MySQL 5.7.22 that collects values from multiple rows into a single JSON array. It is analogous to `GROUP_CONCAT()` but produces a proper JSON-typed array instead of a comma-separated string.

Syntax:

```sql
JSON_ARRAYAGG(expr [ORDER BY ...])
```

## Basic Usage

Given a `tags` table:

```sql
CREATE TABLE post_tags (
  post_id INT,
  tag VARCHAR(50)
);

INSERT INTO post_tags VALUES
(1, 'mysql'), (1, 'json'), (1, 'database'),
(2, 'python'), (2, 'api');
```

Aggregate tags per post into a JSON array:

```sql
SELECT
  post_id,
  JSON_ARRAYAGG(tag) AS tags
FROM post_tags
GROUP BY post_id;
```

Result:

```text
+---------+-------------------------------+
| post_id | tags                          |
+---------+-------------------------------+
|       1 | ["mysql", "json", "database"] |
|       2 | ["python", "api"]             |
+---------+-------------------------------+
```

## Using ORDER BY Inside JSON_ARRAYAGG()

MySQL 8.0 supports `ORDER BY` within the aggregate to control element order:

```sql
SELECT
  post_id,
  JSON_ARRAYAGG(tag ORDER BY tag ASC) AS sorted_tags
FROM post_tags
GROUP BY post_id;
```

This ensures the resulting array is alphabetically ordered regardless of storage order.

## Aggregating Objects with JSON_OBJECT()

Combine `JSON_ARRAYAGG()` with `JSON_OBJECT()` to build arrays of objects:

```sql
SELECT
  department_id,
  JSON_ARRAYAGG(
    JSON_OBJECT('id', id, 'name', name, 'salary', salary)
    ORDER BY name
  ) AS employees
FROM employees
GROUP BY department_id;
```

This is a powerful pattern for building nested API response structures directly in SQL.

## Handling NULL Values

`JSON_ARRAYAGG()` includes `NULL` values in the resulting array:

```sql
SELECT JSON_ARRAYAGG(value) FROM (
  SELECT 1 AS value UNION ALL
  SELECT NULL UNION ALL
  SELECT 3
) t;
-- Result: [1, null, 3]
```

To exclude nulls, use a `WHERE` clause or `CASE`:

```sql
SELECT JSON_ARRAYAGG(value) FROM (
  SELECT 1 AS value UNION ALL
  SELECT NULL UNION ALL
  SELECT 3
) t
WHERE value IS NOT NULL;
-- Result: [1, 3]
```

## Practical Example: One-to-Many Relationships

Flatten a one-to-many relationship into a single query result:

```sql
SELECT
  o.order_id,
  o.customer_id,
  JSON_ARRAYAGG(
    JSON_OBJECT(
      'product_id', oi.product_id,
      'quantity', oi.quantity,
      'price', oi.unit_price
    )
  ) AS items
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.customer_id;
```

This eliminates multiple round trips and produces a document-ready result.

## Performance Considerations

- `JSON_ARRAYAGG()` materializes all values in memory before building the array. For large result sets, this can increase memory usage.
- If you only need a comma-separated string, `GROUP_CONCAT()` is more efficient.
- Adding `ORDER BY` inside the aggregate adds sorting overhead - only use it when ordering is required.

## Summary

`JSON_ARRAYAGG()` transforms grouped rows into JSON arrays within a single query. Combined with `JSON_OBJECT()`, it enables efficient construction of nested JSON documents from relational data, reducing application-side processing and network overhead.
