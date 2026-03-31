# How to Use Comparison Operators in MySQL WHERE Clause

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Comparison Operator, WHERE Clause

Description: Learn how to use MySQL comparison operators including =, !=, <, >, <=, >= and <=> in WHERE clauses to filter query results precisely.

---

Comparison operators are the building blocks of `WHERE` clauses in MySQL. They let you filter rows by comparing column values to literals, other columns, or subquery results. Understanding each operator and its behavior - especially around NULL - prevents subtle bugs.

## Basic Equality and Inequality

```sql
-- Equal to
SELECT * FROM employees WHERE department = 'Engineering';

-- Not equal (both forms are equivalent)
SELECT * FROM employees WHERE salary != 50000;
SELECT * FROM employees WHERE salary <> 50000;
```

## Greater Than and Less Than

```sql
-- Greater than
SELECT * FROM orders WHERE total > 100.00;

-- Less than
SELECT * FROM orders WHERE total < 100.00;

-- Greater than or equal to
SELECT * FROM products WHERE stock_qty >= 10;

-- Less than or equal to
SELECT * FROM products WHERE price <= 29.99;
```

## Combining Comparisons

You can combine multiple comparisons with `AND` and `OR`:

```sql
SELECT product_name, price, stock_qty
FROM products
WHERE price >= 10.00
  AND price <= 50.00
  AND stock_qty > 0;
```

## Comparing Columns to Each Other

Comparison operators work between two columns, not just a column and a literal:

```sql
-- Find orders where the shipped date is after the required date (late shipments)
SELECT order_id, required_date, shipped_date
FROM orders
WHERE shipped_date > required_date;
```

## NULL-Safe Equality with <=>

Regular `=` returns NULL (not TRUE or FALSE) when either operand is NULL. The NULL-safe equality operator `<=>` returns TRUE when both operands are NULL:

```sql
-- Regular = returns NULL for NULL comparisons
SELECT NULL = NULL;    -- Returns NULL
SELECT NULL = 'text';  -- Returns NULL

-- NULL-safe <=> works correctly
SELECT NULL <=> NULL;    -- Returns 1 (TRUE)
SELECT NULL <=> 'text';  -- Returns 0 (FALSE)

-- Practical use: find rows where two nullable columns match
SELECT *
FROM shipments
WHERE delivered_at <=> expected_at;
```

## Comparing Dates and Strings

```sql
-- Date comparison
SELECT * FROM events
WHERE event_date >= '2025-01-01'
  AND event_date < '2026-01-01';

-- String comparison (alphabetical/collation-based)
SELECT * FROM customers
WHERE last_name >= 'M';
```

## Using Comparisons with Subqueries

```sql
-- Find products priced above the average
SELECT product_name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

## Operator Precedence

Comparison operators have lower precedence than arithmetic but higher than logical operators. Use parentheses to make intent explicit:

```sql
-- Without parentheses: potentially confusing
SELECT * FROM sales WHERE qty * price > 1000 AND discount < 0.2;

-- With parentheses: clear intent
SELECT * FROM sales WHERE (qty * price) > 1000 AND (discount < 0.2);
```

## Summary

MySQL's comparison operators (`=`, `!=`/`<>`, `<`, `>`, `<=`, `>=`) work as expected for numbers, strings, and dates. The key edge case is NULL: comparisons with NULL using standard operators always return NULL rather than TRUE or FALSE. Use `IS NULL` / `IS NOT NULL` to check for NULL, or use the null-safe `<=>` operator when you need to compare nullable columns for equality. For date ranges, prefer open-ended comparisons (`>=` and `<`) over `BETWEEN` when end-date exclusivity matters.
