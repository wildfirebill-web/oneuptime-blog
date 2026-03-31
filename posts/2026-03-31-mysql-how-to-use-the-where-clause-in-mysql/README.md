# How to Use the WHERE Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, WHERE Clause, Filtering, SQL Basics, Querying

Description: Learn how to use the WHERE clause in MySQL to filter rows using comparison operators, logical operators, BETWEEN, IN, LIKE, and NULL checks.

---

## Introduction to WHERE

The `WHERE` clause filters rows returned by a `SELECT`, `UPDATE`, or `DELETE` statement. Only rows that satisfy the condition are included in the result.

```sql
-- Basic WHERE with equality
SELECT * FROM orders WHERE status = 'pending';

-- Works with UPDATE and DELETE too
UPDATE orders SET status = 'cancelled' WHERE id = 42;
DELETE FROM logs WHERE created_at < '2025-01-01';
```

## Comparison Operators

```sql
-- Equals, not equals
SELECT * FROM products WHERE category = 'Electronics';
SELECT * FROM products WHERE category != 'Books';
SELECT * FROM products WHERE category <> 'Books';  -- Same as !=

-- Greater than, less than
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price < 50;
SELECT * FROM products WHERE price >= 100;
SELECT * FROM products WHERE price <= 50;
```

## Logical Operators: AND, OR, NOT

```sql
-- AND: both conditions must be true
SELECT * FROM products
WHERE category = 'Electronics' AND price < 500;

-- OR: either condition can be true
SELECT * FROM products
WHERE category = 'Electronics' OR category = 'Computers';

-- NOT: negate a condition
SELECT * FROM products
WHERE NOT category = 'Books';

-- Combining with parentheses for clarity
SELECT * FROM products
WHERE (category = 'Electronics' OR category = 'Computers')
  AND price BETWEEN 100 AND 1000;
```

## BETWEEN Operator

```sql
-- Numeric range (inclusive on both ends)
SELECT * FROM products WHERE price BETWEEN 50 AND 200;

-- Date range
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31';

-- NOT BETWEEN
SELECT * FROM products WHERE price NOT BETWEEN 100 AND 500;
```

## IN and NOT IN

```sql
-- Match any value in a list
SELECT * FROM orders WHERE status IN ('pending', 'processing', 'shipped');

-- Equivalent to multiple OR conditions but cleaner
SELECT * FROM products WHERE category IN ('Electronics', 'Books', 'Clothing');

-- NOT IN: exclude values
SELECT * FROM products WHERE category NOT IN ('Books', 'Stationery');

-- IN with subquery
SELECT * FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders WHERE total > 1000);
```

## LIKE for Pattern Matching

```sql
-- Starts with 'A'
SELECT * FROM customers WHERE first_name LIKE 'A%';

-- Ends with 'son'
SELECT * FROM customers WHERE last_name LIKE '%son';

-- Contains 'tech' anywhere
SELECT * FROM products WHERE name LIKE '%tech%';

-- Exactly 5 characters: use _ for single-char wildcard
SELECT * FROM codes WHERE code LIKE '_____';

-- NOT LIKE
SELECT * FROM products WHERE name NOT LIKE '%refurbished%';
```

## IS NULL and IS NOT NULL

```sql
-- Find rows where a column is NULL
SELECT * FROM customers WHERE phone IS NULL;

-- Find rows where a column is not NULL
SELECT * FROM customers WHERE phone IS NOT NULL;

-- Never use = NULL (this does not work as expected)
-- SELECT * FROM customers WHERE phone = NULL;  -- WRONG, always returns 0 rows
```

## WHERE with Dates

```sql
-- Exact date match
SELECT * FROM orders WHERE DATE(created_at) = '2025-06-15';

-- Using date functions
SELECT * FROM orders WHERE YEAR(created_at) = 2025;
SELECT * FROM orders WHERE MONTH(created_at) = 6;

-- Last 30 days
SELECT * FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY);
```

## WHERE with Multiple Tables (JOINs)

```sql
-- Filter after joining
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total > 500 AND c.country = 'US';
```

## WHERE vs HAVING

```sql
-- WHERE filters rows before aggregation (cannot use aggregate functions)
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE is_active = 1        -- Filters individual rows
GROUP BY department
HAVING avg_salary > 80000; -- Filters groups after aggregation
```

## Summary

The `WHERE` clause is the primary filtering mechanism in MySQL. Use comparison operators for exact and range checks, `IN` for matching a list of values, `LIKE` for pattern matching, and `IS NULL` for null checks. Combine conditions with `AND`, `OR`, and `NOT`, and use parentheses to control evaluation order. Remember that `WHERE` filters rows before grouping, while `HAVING` filters after aggregation.
