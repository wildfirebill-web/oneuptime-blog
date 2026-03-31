# How to Use CASE Expression in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Conditional, Database, Query

Description: Learn how to use the CASE expression in MySQL to add conditional logic to queries, handle NULL values, and build dynamic column values.

---

## What Is the CASE Expression?

The `CASE` expression in MySQL allows you to add conditional logic directly in SQL queries. It works like an if-else statement and can appear in `SELECT`, `WHERE`, `ORDER BY`, and `GROUP BY` clauses.

MySQL supports two forms: the simple CASE and the searched CASE.

## Simple CASE Syntax

```sql
CASE expression
  WHEN value1 THEN result1
  WHEN value2 THEN result2
  ELSE default_result
END
```

## Searched CASE Syntax

```sql
CASE
  WHEN condition1 THEN result1
  WHEN condition2 THEN result2
  ELSE default_result
END
```

## Basic Example: Labeling Status Codes

```sql
SELECT
  order_id,
  status_code,
  CASE status_code
    WHEN 1 THEN 'Pending'
    WHEN 2 THEN 'Processing'
    WHEN 3 THEN 'Shipped'
    WHEN 4 THEN 'Delivered'
    ELSE 'Unknown'
  END AS status_label
FROM orders;
```

## Searched CASE: Range-Based Logic

```sql
SELECT
  product_name,
  price,
  CASE
    WHEN price < 10 THEN 'Budget'
    WHEN price BETWEEN 10 AND 50 THEN 'Mid-range'
    WHEN price > 50 THEN 'Premium'
    ELSE 'Unpriced'
  END AS price_category
FROM products;
```

## CASE in ORDER BY

Sort by a custom priority order:

```sql
SELECT order_id, status_code
FROM orders
ORDER BY
  CASE status_code
    WHEN 1 THEN 1  -- Pending first
    WHEN 2 THEN 2
    WHEN 3 THEN 3
    ELSE 4
  END;
```

## CASE in WHERE Clause

Apply conditional filters:

```sql
SELECT *
FROM employees
WHERE 1 = CASE
  WHEN department = 'Sales' AND salary > 50000 THEN 1
  WHEN department = 'Engineering' AND salary > 80000 THEN 1
  ELSE 0
END;
```

## CASE in GROUP BY for Custom Aggregation

Count orders by price tier:

```sql
SELECT
  CASE
    WHEN total < 100 THEN 'Small'
    WHEN total BETWEEN 100 AND 500 THEN 'Medium'
    ELSE 'Large'
  END AS order_size,
  COUNT(*) AS num_orders,
  SUM(total) AS revenue
FROM orders
GROUP BY order_size;
```

## CASE with NULL Handling

Handle NULL values explicitly:

```sql
SELECT
  name,
  CASE
    WHEN email IS NULL THEN 'No email'
    ELSE email
  END AS contact_email
FROM users;
```

This is equivalent to using `COALESCE(email, 'No email')`, but `CASE` gives you more control over the fallback logic.

## Nested CASE Expressions

CASE expressions can be nested for more complex logic:

```sql
SELECT
  employee_id,
  CASE
    WHEN department = 'Sales' THEN
      CASE
        WHEN quota_met = 1 THEN 'Bonus eligible'
        ELSE 'Below quota'
      END
    ELSE 'Non-sales'
  END AS bonus_status
FROM employees;
```

## CASE vs IF()

For simple two-branch logic, `IF()` is shorter:

```sql
-- Equivalent
SELECT IF(active = 1, 'Active', 'Inactive') FROM users;
SELECT CASE WHEN active = 1 THEN 'Active' ELSE 'Inactive' END FROM users;
```

Use `CASE` when you have three or more branches or when you need to use the expression in `ORDER BY` or `GROUP BY`.

## Summary

The `CASE` expression is MySQL's primary tool for conditional logic in SQL. Use the simple form for equality checks and the searched form for range or complex conditions. It works in `SELECT`, `WHERE`, `ORDER BY`, and `GROUP BY` clauses, making it highly versatile for building dynamic result sets and custom aggregations.
