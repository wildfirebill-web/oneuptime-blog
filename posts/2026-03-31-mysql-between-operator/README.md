# How to Use the BETWEEN Operator in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, BETWEEN, Query

Description: Learn how to use the MySQL BETWEEN operator to filter rows within a numeric, date, or string range inclusively, and when to use it vs. >= and <=.

---

The `BETWEEN` operator in MySQL tests whether a value falls within a closed range. It is a convenient shorthand for `>= lower AND <= upper` and works with numbers, dates, and strings.

## Basic Numeric Range

```sql
-- Find products priced between $10 and $50 (inclusive)
SELECT product_name, price
FROM products
WHERE price BETWEEN 10.00 AND 50.00;

-- Equivalent to:
SELECT product_name, price
FROM products
WHERE price >= 10.00 AND price <= 50.00;
```

Both queries return the same results. `BETWEEN` is inclusive on both ends.

## Date Range with BETWEEN

```sql
-- Find orders placed in Q1 2025
SELECT order_id, customer_id, total, order_date
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-03-31';
```

Be careful with `DATETIME` columns - `'2025-03-31'` is interpreted as `'2025-03-31 00:00:00'`, which excludes orders placed on March 31 after midnight:

```sql
-- Safe approach for DATETIME columns
SELECT order_id, order_date
FROM orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2025-04-01';
```

## NOT BETWEEN

Use `NOT BETWEEN` to exclude a range:

```sql
-- Find products outside the $10-$50 price range
SELECT product_name, price
FROM products
WHERE price NOT BETWEEN 10.00 AND 50.00;
```

## String Range with BETWEEN

`BETWEEN` works with strings using collation-based ordering:

```sql
-- Find customers with last names starting A through M
SELECT customer_id, last_name, first_name
FROM customers
WHERE last_name BETWEEN 'A' AND 'Mzzz';
```

String `BETWEEN` behavior depends on the column collation. For case-insensitive collations like `utf8mb4_general_ci`, 'a' and 'A' are treated as equal.

## Using BETWEEN with Integer IDs

```sql
-- Process a batch of records by ID range
SELECT id, status, created_at
FROM job_queue
WHERE id BETWEEN 1000 AND 1999
  AND status = 'pending';
```

This is common in batch processing where you partition work by ID ranges.

## BETWEEN in HAVING Clause

```sql
-- Find departments with average salary between 50000 and 80000
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING avg_salary BETWEEN 50000 AND 80000;
```

## Performance Considerations

```sql
-- BETWEEN uses indexes efficiently for range scans
EXPLAIN SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31';
```

If an index exists on `order_date`, MySQL will use a range scan, which is efficient. Check the `EXPLAIN` output to confirm index usage - you want to see `range` in the `type` column.

## Summary

The `BETWEEN` operator is a readable shorthand for inclusive range comparisons in MySQL. It works with numbers, dates, and strings, and uses indexes efficiently for range scans. The key things to remember are that both bounds are inclusive, and for `DATETIME` columns the upper bound is treated as midnight of that date - so prefer open-ended `< next_day` comparisons when you need to include a full calendar day. Use `NOT BETWEEN` to exclude a range.
