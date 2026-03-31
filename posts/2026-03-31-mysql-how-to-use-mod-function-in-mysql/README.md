# How to Use MOD() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MOD, Modulo, Math Function, SQL

Description: Learn how to use the MOD() function in MySQL to compute the remainder of division, with practical examples for odd/even checks, cycling, and pagination.

---

## What Is MOD?

`MOD(N, M)` returns the remainder when N is divided by M. It is equivalent to the `%` operator in MySQL. The result has the same sign as the dividend (N). MOD supports integer and floating-point arguments.

```sql
SELECT MOD(10, 3);   -- Returns 1  (10 = 3*3 + 1)
SELECT MOD(15, 5);   -- Returns 0  (15 = 5*3 + 0)
SELECT MOD(7, 2);    -- Returns 1  (7 = 2*3 + 1)
SELECT 10 % 3;       -- Returns 1  (same as MOD)
```

## Basic Examples

```sql
-- Integer modulo
SELECT MOD(17, 5);    -- Returns 2
SELECT MOD(100, 7);   -- Returns 2
SELECT MOD(0, 5);     -- Returns 0

-- Floating-point modulo
SELECT MOD(10.5, 3);  -- Returns 1.5
SELECT MOD(10, 3.5);  -- Returns 3

-- Negative numbers: sign follows the dividend
SELECT MOD(-10, 3);   -- Returns -1
SELECT MOD(10, -3);   -- Returns 1

-- Division by zero
SELECT MOD(10, 0);    -- Returns NULL
```

## Even and Odd Check

```sql
-- Find all even-numbered IDs
SELECT id, name FROM products WHERE MOD(id, 2) = 0;

-- Find all odd-numbered rows
SELECT id, name FROM products WHERE MOD(id, 2) = 1;

-- Label rows as odd or even
SELECT id, name,
    CASE MOD(id, 2)
        WHEN 0 THEN 'Even'
        ELSE 'Odd'
    END AS parity
FROM products;
```

## Cycling and Rotation

```sql
-- Assign users to 3 groups in round-robin fashion
SELECT id, username,
    MOD(id - 1, 3) + 1 AS group_assignment
FROM users;
-- id=1 -> group 1, id=2 -> group 2, id=3 -> group 3,
-- id=4 -> group 1, id=5 -> group 2, ...

-- Distribute orders across 5 warehouses
SELECT order_id,
    CONCAT('WH-', LPAD(MOD(order_id, 5) + 1, 2, '0')) AS assigned_warehouse
FROM orders;
```

## Row Sampling

```sql
-- Select every 10th row for sampling (efficient approximation)
SELECT * FROM events WHERE MOD(id, 10) = 0;

-- Get a 20% random sample using MOD
SELECT * FROM customers WHERE MOD(id, 5) = 0;

-- Sample based on a hash for more even distribution
SELECT * FROM users WHERE MOD(CRC32(uuid), 100) < 10;
-- Approximately 10% sample
```

## Time-Based Cycling

```sql
-- Assign a scheduled slot based on user ID (for scheduled jobs)
-- Each slot runs tasks every 60 seconds for users in that bucket
SELECT user_id,
    MOD(user_id, 60) AS second_offset
FROM users;

-- Color-code rows by position in groups of 5
SELECT
    id,
    name,
    MOD(ROW_NUMBER() OVER (ORDER BY id) - 1, 5) AS color_band
FROM products;
```

## Pagination Calculations

```sql
-- Calculate which page a row falls on (100 rows per page)
SELECT id, name,
    FLOOR((ROW_NUMBER() OVER (ORDER BY id) - 1) / 100) + 1 AS page_number,
    MOD(ROW_NUMBER() OVER (ORDER BY id) - 1, 100) + 1 AS position_on_page
FROM articles;

-- Count rows and pages
SELECT
    COUNT(*) AS total_rows,
    CEIL(COUNT(*) / 100) AS total_pages,
    MOD(COUNT(*), 100) AS rows_on_last_page
FROM articles;
```

## Working with Dates

```sql
-- Find events that fall on weeks that are multiples of 2 (bi-weekly)
SELECT event_date, event_name
FROM events
WHERE MOD(WEEK(event_date), 2) = 0;

-- Find the day offset within a 7-day cycle starting from a reference date
SELECT
    order_date,
    MOD(DATEDIFF(order_date, '2025-01-01'), 7) AS cycle_day
FROM orders;
```

## MOD in Stored Procedures

```sql
DELIMITER //
CREATE PROCEDURE distribute_tasks(IN num_workers INT)
BEGIN
    UPDATE tasks
    SET assigned_worker = MOD(id - 1, num_workers) + 1
    WHERE assigned_worker IS NULL
    ORDER BY priority DESC, created_at ASC;
END //
DELIMITER ;

CALL distribute_tasks(4);  -- Distribute among 4 workers
```

## Summary

The `MOD()` function computes the remainder of integer or floating-point division. It is essential for even/odd checks, round-robin distribution, row sampling, and cycling patterns. Use `MOD(id, N) = 0` to select every Nth row, and `MOD(value - 1, N) + 1` to create cycling sequences from 1 to N. Remember that the result's sign matches the dividend, and division by zero returns NULL.
