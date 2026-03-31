# How to Use the HAVING Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, HAVING, GROUP BY, Aggregation, SQL Filtering

Description: Learn how to use the HAVING clause in MySQL to filter grouped results based on aggregate function conditions after a GROUP BY operation.

---

## What Is the HAVING Clause?

The `HAVING` clause filters groups produced by `GROUP BY`, similar to how `WHERE` filters individual rows. It is evaluated after aggregation, so you can use aggregate functions like `SUM()`, `COUNT()`, `AVG()`, `MIN()`, and `MAX()` in the condition.

```sql
-- Show only departments where average salary exceeds 80000
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 80000;
```

## HAVING vs WHERE

The key distinction: `WHERE` filters rows before grouping, `HAVING` filters groups after grouping.

```sql
-- WHERE: filter individual rows first, then group
SELECT department, COUNT(*) AS active_count
FROM employees
WHERE is_active = 1          -- Filters rows before grouping
GROUP BY department;

-- HAVING: filter aggregated groups
SELECT department, COUNT(*) AS active_count
FROM employees
WHERE is_active = 1
GROUP BY department
HAVING active_count > 5;     -- Filters groups after aggregation

-- Using both together
SELECT department, AVG(salary) AS avg_sal
FROM employees
WHERE hire_date >= '2020-01-01'   -- Only recently hired
GROUP BY department
HAVING avg_sal BETWEEN 70000 AND 120000;  -- Only moderate salary depts
```

## Practical Examples

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    amount DECIMAL(10,2),
    order_date DATE,
    status VARCHAR(20)
);
```

### Find High-Value Customers

```sql
-- Customers with total spending over $1000
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING total_spent > 1000
ORDER BY total_spent DESC;
```

### Find Frequently Ordered Products

```sql
-- Products ordered more than 10 times
SELECT product_id, COUNT(*) AS order_count
FROM orders
GROUP BY product_id
HAVING order_count > 10
ORDER BY order_count DESC;
```

### Find Customers with Multiple Orders in a Month

```sql
SELECT
    customer_id,
    YEAR(order_date) AS yr,
    MONTH(order_date) AS mo,
    COUNT(*) AS monthly_orders
FROM orders
GROUP BY customer_id, YEAR(order_date), MONTH(order_date)
HAVING monthly_orders > 3;
```

## HAVING with Multiple Conditions

```sql
-- Customers who placed more than 5 orders AND spent over $500
SELECT customer_id, COUNT(*) AS num_orders, SUM(amount) AS total
FROM orders
GROUP BY customer_id
HAVING num_orders > 5 AND total > 500;

-- Products with either high order count or high average amount
SELECT product_id, COUNT(*) AS cnt, AVG(amount) AS avg_amt
FROM orders
GROUP BY product_id
HAVING cnt > 20 OR avg_amt > 250;
```

## HAVING with Aggregate Functions Directly

You can reference aggregate functions directly in HAVING rather than column aliases (though aliases are cleaner):

```sql
-- Both syntaxes work
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING COUNT(*) >= 3;  -- Direct aggregate function

-- Or using the alias (MySQL supports this)
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING headcount >= 3;  -- Alias reference
```

## HAVING Without GROUP BY

In MySQL, `HAVING` can be used without `GROUP BY` - it treats the entire result set as a single group:

```sql
-- Check if the table has any rows with high salary
SELECT AVG(salary) AS overall_avg
FROM employees
HAVING overall_avg > 75000;

-- Returns a row only if the condition is true, otherwise returns nothing
```

## Performance Considerations

```sql
-- Filter early with WHERE when possible to reduce rows before grouping
-- BAD: This groups all rows, then filters
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id
HAVING MIN(order_date) > '2024-01-01';

-- BETTER: Use WHERE to filter dates first
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE order_date > '2024-01-01'
GROUP BY customer_id;
-- Note: These are not equivalent if you want "customers whose FIRST order is after 2024"
-- vs "total spend from orders after 2024" - understand the difference!
```

## Summary

The `HAVING` clause is essential for filtering aggregated data in MySQL. Unlike `WHERE`, it runs after `GROUP BY` and can reference aggregate functions. Use it to find groups that meet thresholds like "customers who spent more than $1000" or "departments with more than 10 employees." Combine `WHERE` and `HAVING` in the same query to pre-filter rows and then post-filter groups for maximum efficiency.
