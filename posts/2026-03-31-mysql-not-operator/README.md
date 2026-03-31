# How to Use the NOT Operator in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, NOT Operator, WHERE Clause

Description: Learn how to use the MySQL NOT operator to negate conditions in WHERE clauses, including NOT IN, NOT LIKE, NOT BETWEEN, and NOT EXISTS patterns.

---

The `NOT` operator in MySQL negates a boolean expression, turning TRUE into FALSE and FALSE into TRUE. It is commonly used with `IN`, `LIKE`, `BETWEEN`, `EXISTS`, and `NULL` checks to exclude rows matching specific criteria.

## Basic NOT Usage

```sql
-- NOT with a simple comparison
SELECT * FROM employees WHERE NOT (department = 'HR');

-- Equivalent shorter form
SELECT * FROM employees WHERE department != 'HR';
SELECT * FROM employees WHERE department <> 'HR';
```

## NOT IN

Exclude rows where a column matches any value in a list:

```sql
-- Find orders not in these statuses
SELECT order_id, status, total
FROM orders
WHERE status NOT IN ('cancelled', 'refunded', 'draft');

-- With a subquery
SELECT customer_id, name
FROM customers
WHERE customer_id NOT IN (
  SELECT customer_id FROM orders WHERE order_date >= '2025-01-01'
);
```

Important: if the subquery returns any NULL values, `NOT IN` with a subquery returns no rows. Use `NOT EXISTS` instead to avoid this trap.

## NOT LIKE

```sql
-- Exclude test and example email domains
SELECT user_id, email
FROM users
WHERE email NOT LIKE '%@test.%'
  AND email NOT LIKE '%@example.%';
```

## NOT BETWEEN

```sql
-- Find products outside the standard price range
SELECT product_name, price
FROM products
WHERE price NOT BETWEEN 5.00 AND 99.99;
```

## NOT EXISTS

`NOT EXISTS` is safer than `NOT IN` when the subquery might return NULLs:

```sql
-- Find customers who have never placed an order
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

## NOT NULL Check

```sql
-- Find rows where the column IS NOT NULL
SELECT * FROM employees WHERE manager_id IS NOT NULL;

-- NOT with IS NULL (equivalent)
SELECT * FROM employees WHERE NOT (manager_id IS NULL);
```

Note: `NOT NULL` in a column definition context means the column cannot contain NULL values - this is different from using `NOT` with `IS NULL` in a WHERE clause.

## Combining NOT with AND/OR

Be careful with De Morgan's laws when using NOT:

```sql
-- NOT distributes over AND: NOT (A AND B) = (NOT A) OR (NOT B)
SELECT * FROM products
WHERE NOT (category = 'Electronics' AND price > 500);

-- Equivalent to:
SELECT * FROM products
WHERE category != 'Electronics' OR price <= 500;
```

## Performance Note

`NOT IN` and `NOT LIKE '%pattern%'` with leading wildcards both require full table scans in most cases. `NOT EXISTS` can sometimes use indexes more effectively:

```sql
-- Often faster than NOT IN for large datasets
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM blacklist b WHERE b.customer_id = c.customer_id
);
```

## Summary

The MySQL `NOT` operator negates boolean expressions and is most commonly used with `IN`, `LIKE`, `BETWEEN`, `EXISTS`, and `IS NULL` checks. Prefer `NOT EXISTS` over `NOT IN` when working with subqueries, as `NOT IN` returns no results if the subquery includes any NULL values. Keep De Morgan's law in mind when negating compound conditions, and be aware that many `NOT` patterns disable index use and require full table scans.
