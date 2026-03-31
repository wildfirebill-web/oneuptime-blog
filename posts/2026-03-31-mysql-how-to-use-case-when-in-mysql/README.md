# How to Use CASE WHEN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CASE WHEN, Conditional Logic, SQL, Querying

Description: Learn how to use CASE WHEN expressions in MySQL to add conditional logic to your queries, transforming and categorizing data dynamically.

---

## What Is CASE WHEN?

`CASE WHEN` is MySQL's conditional expression. It evaluates a series of conditions and returns the value for the first condition that is true. Think of it as an if-then-else construct embedded in SQL. It can be used in `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, and `UPDATE` statements.

There are two forms: **searched CASE** and **simple CASE**.

## Searched CASE Expression

```sql
-- Categorize employees by salary range
SELECT
    first_name,
    salary,
    CASE
        WHEN salary < 50000 THEN 'Entry Level'
        WHEN salary < 80000 THEN 'Mid Level'
        WHEN salary < 120000 THEN 'Senior Level'
        ELSE 'Executive'
    END AS salary_band
FROM employees;
```

## Simple CASE Expression

```sql
-- Map status codes to readable labels
SELECT
    id,
    CASE status
        WHEN 'A' THEN 'Active'
        WHEN 'I' THEN 'Inactive'
        WHEN 'P' THEN 'Pending'
        ELSE 'Unknown'
    END AS status_label
FROM accounts;
```

## CASE WHEN in Aggregations

One of the most powerful uses of `CASE WHEN` is inside aggregate functions to perform conditional counting and summing:

```sql
-- Count orders by status in a single row
SELECT
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;

-- Sum revenue only for completed orders per region
SELECT
    region,
    SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS completed_revenue,
    SUM(CASE WHEN status = 'returned' THEN amount ELSE 0 END) AS returned_revenue
FROM orders
GROUP BY region;
```

## CASE WHEN as a Pivot

```sql
-- Pivot monthly sales data into columns
SELECT
    salesperson,
    SUM(CASE WHEN MONTH(sale_date) = 1 THEN amount ELSE 0 END) AS jan,
    SUM(CASE WHEN MONTH(sale_date) = 2 THEN amount ELSE 0 END) AS feb,
    SUM(CASE WHEN MONTH(sale_date) = 3 THEN amount ELSE 0 END) AS mar,
    SUM(CASE WHEN MONTH(sale_date) = 4 THEN amount ELSE 0 END) AS apr
FROM sales
WHERE YEAR(sale_date) = 2025
GROUP BY salesperson;
```

## CASE WHEN in ORDER BY

Use `CASE WHEN` in `ORDER BY` to define custom sort orders:

```sql
-- Sort by custom status priority
SELECT id, title, status
FROM tickets
ORDER BY
    CASE status
        WHEN 'critical' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        WHEN 'low' THEN 4
        ELSE 5
    END ASC,
    created_at DESC;
```

## CASE WHEN in WHERE Clause

```sql
-- Conditional filtering based on a parameter
SET @filter_type = 'high_value';

SELECT * FROM orders
WHERE 1 = CASE
    WHEN @filter_type = 'high_value' AND amount > 1000 THEN 1
    WHEN @filter_type = 'recent' AND order_date > DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 1
    WHEN @filter_type = 'all' THEN 1
    ELSE 0
END;
```

## CASE WHEN in UPDATE

```sql
-- Apply conditional discount based on order amount
UPDATE orders
SET discount_pct = CASE
    WHEN amount > 1000 THEN 0.15
    WHEN amount > 500  THEN 0.10
    WHEN amount > 200  THEN 0.05
    ELSE 0
END;

-- Conditional status transition
UPDATE tasks
SET status = CASE
    WHEN due_date < CURDATE() AND status = 'pending' THEN 'overdue'
    WHEN completed_at IS NOT NULL THEN 'done'
    ELSE status
END;
```

## Nested CASE WHEN

```sql
-- Nested CASE for multi-dimensional categorization
SELECT
    product_name,
    category,
    price,
    CASE category
        WHEN 'Electronics' THEN
            CASE
                WHEN price > 1000 THEN 'Premium Electronics'
                ELSE 'Standard Electronics'
            END
        WHEN 'Clothing' THEN
            CASE
                WHEN price > 200 THEN 'Luxury Clothing'
                ELSE 'Regular Clothing'
            END
        ELSE 'Other'
    END AS product_tier
FROM products;
```

## Summary

`CASE WHEN` is a versatile conditional expression in MySQL that brings if-then-else logic directly into SQL. Use it in `SELECT` to derive new columns, inside aggregate functions for conditional counting and pivot-style reports, in `ORDER BY` for custom sorting, and in `UPDATE` to apply conditional changes. Both the searched form (full boolean conditions) and the simple form (equality comparison) are supported.
