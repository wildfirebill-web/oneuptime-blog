# How to Use the GROUP BY Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, GROUP BY, Aggregation, SQL, Analytics

Description: Learn how to use the GROUP BY clause in MySQL to aggregate rows into groups and compute summary statistics with functions like COUNT, SUM, and AVG.

---

## What Is GROUP BY?

The `GROUP BY` clause groups rows that have the same values in specified columns into summary rows. It is almost always used with aggregate functions like `COUNT()`, `SUM()`, `AVG()`, `MIN()`, and `MAX()`.

```sql
-- Count orders per customer
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;
```

## Sample Data Setup

```sql
CREATE TABLE sales (
    id INT AUTO_INCREMENT PRIMARY KEY,
    salesperson VARCHAR(50),
    region VARCHAR(50),
    product VARCHAR(50),
    amount DECIMAL(10, 2),
    sale_date DATE
);

INSERT INTO sales (salesperson, region, product, amount, sale_date) VALUES
('Alice', 'North', 'Widget A', 250.00, '2025-01-05'),
('Bob', 'South', 'Widget B', 180.00, '2025-01-07'),
('Alice', 'North', 'Widget C', 320.00, '2025-01-10'),
('Carol', 'East', 'Widget A', 410.00, '2025-01-12'),
('Bob', 'South', 'Widget A', 225.00, '2025-02-03'),
('Carol', 'East', 'Widget B', 390.00, '2025-02-08'),
('Alice', 'North', 'Widget B', 175.00, '2025-02-14');
```

## Basic GROUP BY with Aggregate Functions

```sql
-- Total sales per salesperson
SELECT salesperson, SUM(amount) AS total_sales
FROM sales
GROUP BY salesperson;

-- Count of sales per region
SELECT region, COUNT(*) AS num_sales
FROM sales
GROUP BY region;

-- Average sale amount per product
SELECT product, AVG(amount) AS avg_amount
FROM sales
GROUP BY product;

-- Min and max sale per region
SELECT region, MIN(amount) AS min_sale, MAX(amount) AS max_sale
FROM sales
GROUP BY region;
```

## GROUP BY with Multiple Columns

```sql
-- Group by both salesperson and region
SELECT salesperson, region, SUM(amount) AS total_sales
FROM sales
GROUP BY salesperson, region
ORDER BY total_sales DESC;

-- Group by year and month
SELECT
    YEAR(sale_date) AS yr,
    MONTH(sale_date) AS mo,
    SUM(amount) AS monthly_total
FROM sales
GROUP BY YEAR(sale_date), MONTH(sale_date)
ORDER BY yr, mo;
```

## Using GROUP BY with WHERE

`WHERE` filters rows before they are grouped:

```sql
-- Only consider sales > 200, then group
SELECT salesperson, COUNT(*) AS big_deals
FROM sales
WHERE amount > 200
GROUP BY salesperson;

-- Sales in Q1 2025 per region
SELECT region, SUM(amount) AS q1_total
FROM sales
WHERE sale_date BETWEEN '2025-01-01' AND '2025-03-31'
GROUP BY region;
```

## GROUP BY with HAVING

`HAVING` filters groups after aggregation:

```sql
-- Only show salespersons with total sales > 500
SELECT salesperson, SUM(amount) AS total_sales
FROM sales
GROUP BY salesperson
HAVING total_sales > 500;

-- Regions with more than 2 sales
SELECT region, COUNT(*) AS num_sales
FROM sales
GROUP BY region
HAVING num_sales > 2;
```

## GROUP BY with ORDER BY and LIMIT

```sql
-- Top 3 salespersons by total revenue
SELECT salesperson, SUM(amount) AS total_sales
FROM sales
GROUP BY salesperson
ORDER BY total_sales DESC
LIMIT 3;

-- Bottom performer by average deal size
SELECT salesperson, AVG(amount) AS avg_deal
FROM sales
GROUP BY salesperson
ORDER BY avg_deal ASC
LIMIT 1;
```

## GROUP BY with Expressions

```sql
-- Group by computed expressions
SELECT
    CASE
        WHEN amount < 200 THEN 'Small'
        WHEN amount < 350 THEN 'Medium'
        ELSE 'Large'
    END AS deal_size,
    COUNT(*) AS count,
    SUM(amount) AS total
FROM sales
GROUP BY deal_size;

-- Group by year extracted from date
SELECT YEAR(sale_date) AS sale_year, SUM(amount) AS yearly_total
FROM sales
GROUP BY YEAR(sale_date);
```

## GROUP_CONCAT - Aggregating Strings

```sql
-- List products sold per salesperson as a comma-separated string
SELECT salesperson, GROUP_CONCAT(DISTINCT product ORDER BY product SEPARATOR ', ') AS products_sold
FROM sales
GROUP BY salesperson;
```

## Summary

The `GROUP BY` clause is the core of analytical queries in MySQL. Pair it with aggregate functions like `SUM()`, `COUNT()`, `AVG()`, `MIN()`, and `MAX()` to summarize data. Use `WHERE` to pre-filter individual rows before grouping, and `HAVING` to post-filter groups based on aggregate results. Combine with `ORDER BY` and `LIMIT` to find top or bottom performers.
