# How to Use CASE WHEN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Case When, Conditional Logic, Sql, Database

Description: Learn how to use CASE WHEN expressions in MySQL for conditional logic in SELECT, WHERE, ORDER BY, and UPDATE statements with practical examples.

---

## Introduction

`CASE WHEN` is MySQL's conditional expression, similar to if-else logic in programming languages. It evaluates conditions and returns different values based on which condition is true. It can be used in `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, and `UPDATE` statements.

## Two Forms of CASE

### Simple CASE (compares a value)

```sql
CASE expression
  WHEN value1 THEN result1
  WHEN value2 THEN result2
  ...
  ELSE default_result
END
```

### Searched CASE (evaluates conditions)

```sql
CASE
  WHEN condition1 THEN result1
  WHEN condition2 THEN result2
  ...
  ELSE default_result
END
```

## Simple CASE Example

```sql
SELECT
  order_id,
  status,
  CASE status
    WHEN 'pending'   THEN 'Awaiting Processing'
    WHEN 'shipped'   THEN 'On the Way'
    WHEN 'delivered' THEN 'Completed'
    ELSE 'Unknown Status'
  END AS status_label
FROM orders;
```

## Searched CASE Example

```sql
SELECT
  employee_id,
  salary,
  CASE
    WHEN salary < 40000  THEN 'Junior'
    WHEN salary < 80000  THEN 'Mid-level'
    WHEN salary < 120000 THEN 'Senior'
    ELSE 'Executive'
  END AS level
FROM employees;
```

## CASE WHEN in WHERE Clause

```sql
SELECT id, name, score
FROM students
WHERE
  CASE
    WHEN grade = 'A' THEN score > 90
    WHEN grade = 'B' THEN score > 75
    ELSE score > 60
  END;
```

## CASE WHEN in ORDER BY

Implement custom sort orders using CASE.

```sql
SELECT name, priority
FROM tasks
ORDER BY
  CASE priority
    WHEN 'critical' THEN 1
    WHEN 'high'     THEN 2
    WHEN 'medium'   THEN 3
    WHEN 'low'      THEN 4
    ELSE 5
  END ASC;
```

## CASE WHEN in GROUP BY

```sql
SELECT
  CASE
    WHEN age < 18 THEN 'Minor'
    WHEN age < 65 THEN 'Adult'
    ELSE 'Senior'
  END AS age_group,
  COUNT(*) AS count
FROM users
GROUP BY age_group;
```

## CASE WHEN in UPDATE

```sql
UPDATE products
SET discount_rate =
  CASE
    WHEN category = 'electronics' THEN 0.10
    WHEN category = 'clothing'    THEN 0.20
    WHEN category = 'food'        THEN 0.05
    ELSE 0.00
  END;
```

## CASE WHEN with Aggregate Functions

Use CASE inside aggregate functions for conditional counting.

```sql
SELECT
  department,
  COUNT(CASE WHEN gender = 'M' THEN 1 END) AS male_count,
  COUNT(CASE WHEN gender = 'F' THEN 1 END) AS female_count,
  COUNT(*) AS total
FROM employees
GROUP BY department;
```

This is a common pattern for "pivot table" style reporting.

## CASE WHEN for Conditional SUM

```sql
SELECT
  product_category,
  SUM(CASE WHEN status = 'sold' THEN revenue ELSE 0 END) AS sold_revenue,
  SUM(CASE WHEN status = 'returned' THEN revenue ELSE 0 END) AS returned_revenue
FROM sales
GROUP BY product_category;
```

## Nested CASE WHEN

```sql
SELECT
  name,
  CASE
    WHEN status = 'active' THEN
      CASE
        WHEN score > 90 THEN 'Active - Top Performer'
        ELSE 'Active - Standard'
      END
    ELSE 'Inactive'
  END AS classification
FROM users;
```

## ELSE Clause is Optional

If `ELSE` is omitted and no condition matches, `CASE` returns NULL.

```sql
SELECT
  name,
  CASE status
    WHEN 'active'   THEN 'Active'
    WHEN 'inactive' THEN 'Inactive'
    -- No ELSE: returns NULL for other status values
  END AS status_label
FROM users;
```

## CASE WHEN vs IF()

`IF()` is a MySQL-specific shorthand for simple two-outcome conditions.

```sql
-- IF() for simple conditions
SELECT IF(score > 50, 'Pass', 'Fail') AS result FROM exam;

-- CASE WHEN for multiple conditions
SELECT
  CASE
    WHEN score >= 90 THEN 'A'
    WHEN score >= 80 THEN 'B'
    WHEN score >= 70 THEN 'C'
    ELSE 'F'
  END AS grade
FROM exam;
```

## Summary

`CASE WHEN` is a versatile conditional expression in MySQL used across SELECT, WHERE, ORDER BY, GROUP BY, and UPDATE statements. Use the simple form for equality checks against one value and the searched form for range conditions. It is the standard SQL way to add if-else logic directly into queries.
