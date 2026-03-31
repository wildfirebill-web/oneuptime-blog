# How to Use UNION ALL in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Union All, Set Operations, Sql, Database

Description: Learn how to use UNION ALL in MySQL to combine multiple SELECT results including duplicates, and when to prefer it over UNION for better performance.

---

## Introduction

`UNION ALL` in MySQL combines the results of two or more `SELECT` statements into a single result set, keeping all rows including duplicates. Unlike `UNION`, it does not perform deduplication, which makes it faster and more efficient when you know there are no duplicates or when duplicates are intentionally needed.

## Basic UNION ALL Syntax

```sql
SELECT column1, column2 FROM table1
UNION ALL
SELECT column1, column2 FROM table2;
```

Rules:
- Each SELECT must have the same number of columns.
- Corresponding columns must have compatible data types.
- Column names come from the first SELECT.
- All rows from all queries are included, including duplicates.

## Simple UNION ALL Example

Combine all transactions from two tables, keeping every row:

```sql
SELECT id, amount, transaction_date FROM transactions_2023
UNION ALL
SELECT id, amount, transaction_date FROM transactions_2024;
```

## When to Use UNION ALL vs UNION

| Feature | UNION | UNION ALL |
|---------|-------|-----------|
| Removes duplicates | Yes | No |
| Performance | Slower | Faster |
| Use case | Unique results | All results, or when no duplicates exist |

```sql
-- Use UNION when duplicates must be removed
SELECT email FROM list_a
UNION
SELECT email FROM list_b;

-- Use UNION ALL when duplicates are OK or impossible
SELECT log_entry FROM server1_logs
UNION ALL
SELECT log_entry FROM server2_logs;
```

## UNION ALL for Aggregating Across Tables

Sum up values from multiple partitioned tables:

```sql
SELECT SUM(revenue) AS total_revenue
FROM (
  SELECT revenue FROM sales_q1
  UNION ALL
  SELECT revenue FROM sales_q2
  UNION ALL
  SELECT revenue FROM sales_q3
  UNION ALL
  SELECT revenue FROM sales_q4
) AS all_sales;
```

## UNION ALL with ORDER BY and LIMIT

Apply ORDER BY and LIMIT to the combined result:

```sql
SELECT id, created_at, 'comment' AS type FROM comments
UNION ALL
SELECT id, created_at, 'post' AS type FROM posts
ORDER BY created_at DESC
LIMIT 20;
```

## UNION ALL for Event Logs

Combine logs from multiple sources for unified analysis:

```sql
SELECT user_id, action, timestamp, 'login' AS event_type FROM login_events
UNION ALL
SELECT user_id, action, timestamp, 'purchase' AS event_type FROM purchase_events
UNION ALL
SELECT user_id, action, timestamp, 'logout' AS event_type FROM logout_events
ORDER BY timestamp ASC;
```

## UNION ALL Inside a Subquery

```sql
SELECT department, AVG(salary) AS avg_salary
FROM (
  SELECT department, salary FROM current_employees
  UNION ALL
  SELECT department, salary FROM contract_workers
) AS all_workers
GROUP BY department;
```

## UNION ALL with Three or More Queries

```sql
SELECT product_id, quantity FROM warehouse_a
UNION ALL
SELECT product_id, quantity FROM warehouse_b
UNION ALL
SELECT product_id, quantity FROM warehouse_c;
```

## Adding a Source Identifier

When combining data from different sources, tag each row with its origin:

```sql
SELECT order_id, customer_id, total, 'online' AS channel FROM online_orders
UNION ALL
SELECT order_id, customer_id, total, 'in_store' AS channel FROM store_orders;
```

## Performance Benefits of UNION ALL

`UNION` must create a temporary table to track and remove duplicates. `UNION ALL` bypasses this step entirely.

```sql
EXPLAIN
SELECT name FROM table_a
UNION ALL
SELECT name FROM table_b;
```

You should NOT see "Using temporary" in the output for `UNION ALL` (unlike `UNION`).

## Using UNION ALL for Count Across Tables

```sql
SELECT 'table_a' AS source, COUNT(*) AS row_count FROM table_a
UNION ALL
SELECT 'table_b' AS source, COUNT(*) AS row_count FROM table_b
UNION ALL
SELECT 'table_c' AS source, COUNT(*) AS row_count FROM table_c;
```

## Summary

`UNION ALL` merges multiple SELECT results without removing duplicates, making it faster than `UNION` when deduplication is not required. It is the preferred choice for combining log data, partitioned tables, and aggregations across multiple sources. Always use `UNION ALL` instead of `UNION` when you know there are no duplicates or you explicitly want to keep them.
