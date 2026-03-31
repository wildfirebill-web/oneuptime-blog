# How to Use UNION in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, UNION, Set Operations, SQL, Querying

Description: Learn how to use UNION in MySQL to combine the results of multiple SELECT statements into a single result set while removing duplicate rows.

---

## What Is UNION?

`UNION` combines the results of two or more `SELECT` statements into a single result set. By default, `UNION` removes duplicate rows from the combined result. Each `SELECT` must return the same number of columns, with compatible data types in corresponding positions.

```sql
-- Combine two result sets and remove duplicates
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
```

## Basic UNION Example

```sql
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    city VARCHAR(50)
);

CREATE TABLE leads (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    city VARCHAR(50)
);

-- Get all unique email addresses from both tables
SELECT email FROM customers
UNION
SELECT email FROM leads;
```

## Column Names in UNION

The column names in the result set come from the first SELECT statement:

```sql
-- Column names are taken from the first SELECT
SELECT first_name AS name, email FROM employees
UNION
SELECT company_name, contact_email FROM vendors;
-- Result columns will be named 'name' and 'email'
```

## Ordering UNION Results

Apply `ORDER BY` at the very end, after all UNION components:

```sql
-- Sort the combined result
SELECT name, city, 'customer' AS source FROM customers
UNION
SELECT name, city, 'lead' AS source FROM leads
ORDER BY city, name;
```

## UNION with WHERE Clauses

Each SELECT in a UNION can have its own WHERE clause:

```sql
-- Active customers in New York UNION leads from California
SELECT name, email FROM customers
WHERE city = 'New York' AND status = 'active'
UNION
SELECT name, email FROM leads
WHERE city = 'California';
```

## Practical Use Cases

### Merging Archived and Current Data

```sql
-- Combine current and archived order tables
SELECT id, customer_id, amount, order_date FROM orders
UNION
SELECT id, customer_id, amount, order_date FROM orders_archive
ORDER BY order_date DESC;
```

### Building a Unified Contact List

```sql
-- All contact emails from multiple sources, deduplicated
SELECT email, 'Employee' AS type FROM employees WHERE email IS NOT NULL
UNION
SELECT email, 'Customer' AS type FROM customers WHERE email IS NOT NULL
UNION
SELECT email, 'Vendor' AS type FROM vendors WHERE email IS NOT NULL;
```

### Reporting with Multiple Data Sources

```sql
-- Monthly revenue from two product lines, combined
SELECT
    DATE_FORMAT(sale_date, '%Y-%m') AS month,
    SUM(amount) AS revenue,
    'Software' AS product_line
FROM software_sales
GROUP BY DATE_FORMAT(sale_date, '%Y-%m')

UNION

SELECT
    DATE_FORMAT(sale_date, '%Y-%m') AS month,
    SUM(amount) AS revenue,
    'Hardware' AS product_line
FROM hardware_sales
GROUP BY DATE_FORMAT(sale_date, '%Y-%m')

ORDER BY month, product_line;
```

## UNION with Subqueries

```sql
-- Use subqueries as inputs to UNION
SELECT * FROM (
    SELECT 'Top Customer' AS label, name, SUM(amount) AS total
    FROM orders JOIN customers ON orders.customer_id = customers.id
    GROUP BY customers.id
    ORDER BY total DESC
    LIMIT 1
) top

UNION

SELECT * FROM (
    SELECT 'Bottom Customer' AS label, name, SUM(amount) AS total
    FROM orders JOIN customers ON orders.customer_id = customers.id
    GROUP BY customers.id
    ORDER BY total ASC
    LIMIT 1
) bottom;
```

## Performance Note

`UNION` requires a sort/dedup step to remove duplicates, which adds overhead. If you know there are no duplicates or don't need deduplication, use `UNION ALL` for better performance.

```sql
-- UNION (slower - deduplicates)
SELECT email FROM list_a UNION SELECT email FROM list_b;

-- UNION ALL (faster - no deduplication)
SELECT email FROM list_a UNION ALL SELECT email FROM list_b;
```

## Summary

`UNION` is a powerful set operation that merges result sets from multiple SELECT queries while automatically removing duplicate rows. Ensure that all SELECT statements have the same number of columns with compatible types. Use a single `ORDER BY` at the end of the entire UNION expression. When duplicates are not a concern, prefer `UNION ALL` for better performance.
