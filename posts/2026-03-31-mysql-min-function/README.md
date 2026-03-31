# How to Use the MIN() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MIN Function, Aggregate Function

Description: Learn how to use the MySQL MIN() function to find minimum values in numeric, date, and string columns, with GROUP BY and window function examples.

---

`MIN()` is a MySQL aggregate function that returns the smallest non-NULL value in a set of values. It works with numeric, date, and string columns, returning the minimum value according to the column's data type and collation.

## Basic MIN

```sql
-- Cheapest product price
SELECT MIN(price) AS min_price FROM products;

-- Earliest order date
SELECT MIN(order_date) AS first_order FROM orders;

-- Alphabetically first last name
SELECT MIN(last_name) AS first_alphabetically FROM customers;
```

## MIN with WHERE

```sql
-- Minimum price in the Electronics category
SELECT MIN(price) AS min_electronics_price
FROM products
WHERE category = 'Electronics';

-- Earliest completed order in 2025
SELECT MIN(order_date) AS first_2025_completion
FROM orders
WHERE status = 'completed'
  AND YEAR(order_date) = 2025;
```

## MIN with GROUP BY

```sql
-- Minimum salary per department
SELECT department, MIN(salary) AS min_salary
FROM employees
GROUP BY department
ORDER BY min_salary ASC;

-- Earliest order date per customer
SELECT customer_id, MIN(order_date) AS first_order_date
FROM orders
GROUP BY customer_id;
```

## MIN and MAX Together

```sql
-- Price range per category
SELECT
  category,
  MIN(price) AS lowest_price,
  MAX(price) AS highest_price,
  MAX(price) - MIN(price) AS price_range
FROM products
GROUP BY category
ORDER BY price_range DESC;
```

## Finding the Row with the Minimum Value

`MIN()` returns the value but not the full row. To get the complete row with the minimum value:

```sql
-- Get full product details for the cheapest product
SELECT product_id, product_name, price
FROM products
WHERE price = (SELECT MIN(price) FROM products);

-- Or with a join (more efficient for large tables)
SELECT p.product_id, p.product_name, p.price
FROM products p
INNER JOIN (
  SELECT MIN(price) AS min_price FROM products
) AS m ON p.price = m.min_price;
```

## MIN with HAVING

```sql
-- Find categories where even the cheapest product costs more than $20
SELECT category, MIN(price) AS min_price
FROM products
GROUP BY category
HAVING min_price > 20.00
ORDER BY min_price;
```

## MIN as a Window Function

In MySQL 8.0+, `MIN()` can be used as a window function to see the running minimum:

```sql
-- Running minimum price over ordered product list
SELECT
  product_name,
  price,
  MIN(price) OVER (ORDER BY product_id) AS running_min_price
FROM products;

-- Minimum per partition (department minimum shown for each employee row)
SELECT
  name,
  department,
  salary,
  MIN(salary) OVER (PARTITION BY department) AS dept_min_salary
FROM employees;
```

## NULL Handling

```sql
-- MIN() ignores NULL values
SELECT MIN(discount_amount) FROM orders;
-- Returns minimum of non-NULL discount amounts
```

## Summary

`MIN()` returns the smallest non-NULL value in a column and works with numbers, dates, and strings. Use it with `WHERE` for filtered minimums, with `GROUP BY` for per-group minimums, and with `HAVING` to filter groups by their minimum. To retrieve the full row containing the minimum value, use a subquery or self-join. In MySQL 8.0+, `MIN() OVER (PARTITION BY ...)` shows the group minimum alongside each row without aggregating.
