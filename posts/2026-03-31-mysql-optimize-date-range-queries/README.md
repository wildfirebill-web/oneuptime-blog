# How to Optimize Date Range Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Index, Performance, Optimization

Description: Learn how to optimize MySQL date range queries by using sargable predicates, proper index design, and partitioning for time-series tables.

---

## Why Date Range Queries Are Often Slow

Date range queries are among the most common patterns in MySQL, yet they are frequently written in ways that prevent index use. The main culprits are wrapping date columns in functions and using improper comparison operators.

## The Sargability Problem with Date Functions

Wrapping a date column in a function makes the predicate non-sargable - MySQL must evaluate the function for every row instead of using the index:

```sql
-- Non-sargable: function on indexed column
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2025;
-- type: ALL, key: NULL

-- Non-sargable: DATE() function
EXPLAIN SELECT * FROM orders WHERE DATE(created_at) = '2025-06-15';
-- type: ALL, key: NULL
```

## Write Sargable Date Predicates

Replace function-wrapped conditions with explicit range predicates:

```sql
-- Sargable: range predicate uses the index
CREATE INDEX idx_created ON orders(created_at);

-- Year query rewritten as range
SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';

-- Single day query rewritten as range
SELECT * FROM orders
WHERE created_at >= '2025-06-15' AND created_at < '2025-06-16';

-- Both use index range scan
EXPLAIN SELECT * FROM orders WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
-- type: range, key: idx_created
```

## Use BETWEEN for Inclusive Ranges

```sql
-- BETWEEN is inclusive on both ends
SELECT * FROM orders
WHERE created_at BETWEEN '2025-01-01 00:00:00' AND '2025-12-31 23:59:59';
```

Be careful with DATETIME columns - `BETWEEN '2025-01-01' AND '2025-12-31'` misses rows on December 31 after midnight. Use explicit times or the exclusive upper bound pattern.

## Composite Index for Date + Filter Columns

When filtering on multiple columns including a date range, the column order matters:

```sql
-- Query: filter by status (equality) + date range
SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2025-06-01';

-- Index: equality column first, range column second
CREATE INDEX idx_status_created ON orders(status, created_at);

EXPLAIN SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2025-06-01';
-- type: range, key: idx_status_created
```

Note: if the date range column comes before an equality column in the index, MySQL can only use the index up to and including the range column.

## Partitioning for Large Time-Series Tables

For tables that grow continuously (logs, events, metrics), RANGE partitioning by date is the most scalable long-term approach:

```sql
CREATE TABLE events (
  id BIGINT AUTO_INCREMENT,
  event_type VARCHAR(50),
  created_at DATETIME NOT NULL,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (TO_DAYS(created_at)) (
  PARTITION p2025_q1 VALUES LESS THAN (TO_DAYS('2025-04-01')),
  PARTITION p2025_q2 VALUES LESS THAN (TO_DAYS('2025-07-01')),
  PARTITION p2025_q3 VALUES LESS THAN (TO_DAYS('2025-10-01')),
  PARTITION p2025_q4 VALUES LESS THAN (TO_DAYS('2026-01-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

MySQL uses partition pruning to scan only the relevant partitions for date range queries:

```sql
EXPLAIN SELECT * FROM events WHERE created_at BETWEEN '2025-06-01' AND '2025-06-30';
-- partitions: p2025_q2 (only one partition scanned)
```

## Using Generated Columns for Date Parts

If you frequently query by year, month, or day independently, generated columns allow indexing derived date parts:

```sql
ALTER TABLE orders ADD COLUMN order_date DATE
  AS (DATE(created_at)) STORED;
CREATE INDEX idx_order_date ON orders(order_date);

-- Now an equality query on the date part uses the index
SELECT * FROM orders WHERE order_date = '2025-06-15';
-- type: ref (not range), very efficient
```

## Summary

Date range query optimization in MySQL centers on writing sargable predicates using explicit range comparisons instead of wrapping columns in functions like YEAR(), DATE(), or MONTH(). Add an index on your date column and put equality-filter columns before range columns in composite indexes. For tables that grow continuously, table partitioning by date range provides the best long-term scalability by enabling partition pruning.
