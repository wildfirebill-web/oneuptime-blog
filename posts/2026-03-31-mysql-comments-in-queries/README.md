# How to Use Comments in MySQL Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Comment, Query, Documentation

Description: Learn all MySQL comment syntaxes including single-line, multi-line, and conditional execution comments, with practical use cases for each.

---

## MySQL Comment Syntax Overview

MySQL supports three comment styles, each with distinct use cases.

## Single-Line Comments with --

Use double-dash followed by a space for single-line comments:

```sql
-- Retrieve active users created in the last 30 days
SELECT id, email, created_at
FROM users
WHERE active = 1  -- only active accounts
  AND created_at >= NOW() - INTERVAL 30 DAY;
```

Note: the space after `--` is required. `--comment` without a space is NOT a comment in standard MySQL and will cause a syntax error.

## Single-Line Comments with #

The hash symbol is a MySQL-specific single-line comment:

```sql
# This is a MySQL-specific comment
SELECT COUNT(*) FROM orders; # total order count
```

Hash comments are valid in MySQL but not in standard SQL (ANSI). Avoid them in code that must be portable to other databases.

## Multi-Line Comments with /* */

Use C-style block comments for multi-line explanations or commenting out code blocks:

```sql
/*
  Query: Monthly revenue summary
  Author: analytics team
  Last updated: 2026-03-01
*/
SELECT
  DATE_FORMAT(order_date, '%Y-%m') AS month,
  SUM(total)                        AS revenue,
  COUNT(*)                          AS order_count
FROM orders
/* WHERE status != 'cancelled' -- uncomment to exclude cancellations */
GROUP BY month
ORDER BY month DESC;
```

## Executable Comments for MySQL-Specific Code

MySQL supports a special `/*! ... */` syntax for executable comments. The code inside runs on MySQL but is ignored by other databases:

```sql
-- Create a table with MySQL-specific options, ignored by other SQL parsers
CREATE TABLE events (
  id   INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(255) NOT NULL,
  PRIMARY KEY (id)
) /*!50700 COMPRESSION='zlib' */;

-- Use MySQL 8+ specific syntax conditionally
SELECT /*! STRAIGHT_JOIN */ o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

The number after `/*!` is a minimum MySQL version (e.g., `50700` = MySQL 5.7.0). If the server version is lower, the comment is ignored.

## Optimizer Hints as Comments

MySQL 8 supports optimizer hints embedded in comments:

```sql
-- Force use of a specific index
SELECT /*+ INDEX(orders idx_status_created) */
  id, total
FROM orders
WHERE status = 'pending';

-- Disable block nested loop join
SELECT /*+ NO_BNL(orders, customers) */
  o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Set maximum execution time (milliseconds)
SELECT /*+ MAX_EXECUTION_TIME(5000) */ *
FROM large_table;
```

Optimizer hints use `/*+ ... */` syntax and are case-insensitive.

## Commenting Out Code During Debugging

Block comments are useful for temporarily disabling query parts:

```sql
SELECT id, email
FROM users
WHERE active = 1
  /* AND plan = 'premium' */  -- disabled for testing
  AND created_at > '2026-01-01';
```

## Comments in Stored Procedures

Document stored procedures and functions with multi-line comments:

```sql
DELIMITER $$

CREATE PROCEDURE calculate_monthly_revenue(
  IN p_year INT,
  IN p_month INT
)
BEGIN
  /*
    Calculates total revenue for a given year/month.
    Parameters:
      p_year  - 4-digit year (e.g., 2026)
      p_month - Month number 1-12
  */
  SELECT SUM(total) AS revenue
  FROM orders
  WHERE YEAR(order_date)  = p_year   -- filter by year
    AND MONTH(order_date) = p_month  -- filter by month
    AND status = 'completed';
END$$

DELIMITER ;
```

## Summary

MySQL supports three comment styles: `-- ` for single-line ANSI comments (note the trailing space), `#` for MySQL-specific single-line comments, and `/* */` for multi-line block comments. Use `/*! */` for MySQL-conditional executable code that other parsers ignore, and `/*+ */` for optimizer hints. Good comments explain the "why" behind complex WHERE clauses and join conditions, not just the "what" that the SQL syntax already makes clear.
