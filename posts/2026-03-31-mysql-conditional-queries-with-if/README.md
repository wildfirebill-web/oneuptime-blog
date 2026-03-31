# How to Write Conditional Queries with IF() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IF Function, Conditional, Query, Expression

Description: Learn how to use MySQL's IF() function to write conditional expressions in SELECT, WHERE, and aggregate queries.

---

## What Is the IF() Function

MySQL's `IF()` function evaluates a condition and returns one of two values depending on whether the condition is true or false:

```sql
IF(condition, value_if_true, value_if_false)
```

It is similar to a ternary operator in programming languages and can be used anywhere an expression is valid.

## Basic Usage in SELECT

Label orders based on amount:

```sql
SELECT
  id,
  amount,
  IF(amount >= 100, 'Large', 'Small') AS order_size
FROM orders;
```

Flag users based on activity:

```sql
SELECT
  id,
  username,
  IF(last_login > NOW() - INTERVAL 30 DAY, 'Active', 'Inactive') AS status
FROM users;
```

## Using IF() in a WHERE Clause

`IF()` can be used in a `WHERE` clause to apply different filter logic:

```sql
SELECT id, name, price
FROM products
WHERE price > IF(@min_price IS NULL, 0, @min_price);
```

## Nested IF() for Multiple Conditions

You can nest `IF()` calls for multiple branches:

```sql
SELECT
  id,
  score,
  IF(score >= 90, 'A',
    IF(score >= 80, 'B',
      IF(score >= 70, 'C', 'F')
    )
  ) AS grade
FROM test_results;
```

For more than two branches, `CASE WHEN` is more readable:

```sql
SELECT
  id,
  score,
  CASE
    WHEN score >= 90 THEN 'A'
    WHEN score >= 80 THEN 'B'
    WHEN score >= 70 THEN 'C'
    ELSE 'F'
  END AS grade
FROM test_results;
```

## Conditional Aggregation with IF()

`IF()` is commonly used inside aggregate functions to count or sum conditionally:

```sql
SELECT
  department,
  COUNT(*) AS total_employees,
  SUM(IF(salary > 70000, 1, 0)) AS high_earners,
  SUM(IF(salary <= 70000, 1, 0)) AS regular_earners
FROM employees
GROUP BY department;
```

This avoids multiple subqueries and is often more efficient.

## Using IF() in UPDATE Statements

Apply different values to different rows in a single pass:

```sql
UPDATE products
SET price = IF(category = 'Electronics', price * 0.9, price * 0.95);
```

## IF() vs CASE WHEN

`IF()` is concise for simple two-way conditions. Use `CASE WHEN` when you have three or more branches or need to check multiple columns:

| Scenario              | Recommended |
|-----------------------|-------------|
| Binary condition      | IF()        |
| Multiple conditions   | CASE WHEN   |
| NULL handling         | IFNULL() / COALESCE() |

## Summary

MySQL's `IF()` function enables inline conditional logic within queries, making it easy to label, classify, or conditionally aggregate data without multiple queries. For simple true-or-false decisions it is concise and readable. When conditions grow complex, switch to `CASE WHEN` for clarity.
