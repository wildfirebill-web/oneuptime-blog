# What Is a MySQL Index

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Query Optimization

Description: Learn what a MySQL index is, how B-tree indexes work, how to create and use indexes effectively, and when indexes improve or hurt performance.

---

A MySQL index is a data structure that allows the database engine to find rows without scanning the entire table. Without indexes, every query that filters, sorts, or joins data requires a full table scan - reading every row. With indexes, MySQL can jump directly to matching rows, making queries orders of magnitude faster on large tables.

## How B-Tree Indexes Work

The default MySQL index type is a B-tree (balanced tree). It stores column values in sorted order, allowing MySQL to locate a specific value in O(log n) time rather than O(n) for a full table scan.

```sql
-- Without index: full scan of 10 million rows
SELECT * FROM orders WHERE customer_id = 42;
-- type: ALL in EXPLAIN (full table scan)

-- Add an index
CREATE INDEX idx_customer ON orders (customer_id);

-- With index: direct lookup
SELECT * FROM orders WHERE customer_id = 42;
-- type: ref in EXPLAIN (index lookup)
```

## Creating Indexes

```sql
-- Single-column index
CREATE INDEX idx_email ON users (email);

-- Unique index
CREATE UNIQUE INDEX uk_email ON users (email);

-- Composite (multi-column) index
CREATE INDEX idx_customer_status ON orders (customer_id, status);

-- Index on table creation
CREATE TABLE products (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  sku   VARCHAR(50) NOT NULL,
  name  VARCHAR(255),
  price DECIMAL(10,2),
  INDEX idx_price (price),
  UNIQUE KEY uk_sku (sku)
) ENGINE=InnoDB;
```

## Checking Index Usage with EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending'\G
-- key: idx_customer_status  (index being used)
-- rows: 3                   (estimated rows examined)
-- Extra: Using index condition
```

## Index Selectivity

Indexes are most effective on high-cardinality columns (many distinct values). An index on a boolean column with only two values provides little benefit.

```sql
-- Check cardinality of potential index columns
SELECT
  COUNT(DISTINCT customer_id) AS cust_cardinality,
  COUNT(DISTINCT status)      AS status_cardinality,
  COUNT(*)                    AS total_rows
FROM orders;
-- customer_id: 50000 distinct values (high cardinality - good index candidate)
-- status: 4 distinct values (low cardinality - index less useful alone)
```

## When Indexes Are Not Used

MySQL skips an index when the optimizer estimates a full scan is faster (typically when the query matches a large percentage of rows). Certain patterns also prevent index use:

```sql
-- Function on indexed column prevents index use
SELECT * FROM users WHERE YEAR(created_at) = 2025; -- no index
-- Instead use range:
SELECT * FROM users WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';

-- Leading wildcard prevents index use
SELECT * FROM users WHERE email LIKE '%example.com'; -- no index
-- Trailing wildcard uses index:
SELECT * FROM users WHERE email LIKE 'alice%'; -- uses index
```

## Composite Index Column Order

For a composite index, the leftmost prefix rule applies. The index is useful when queries filter on the leading columns.

```sql
CREATE INDEX idx_cust_date ON orders (customer_id, created_at);

-- Uses index (leading column present)
SELECT * FROM orders WHERE customer_id = 42;

-- Uses index (both columns)
SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2025-01-01';

-- Does NOT use index (leading column missing)
SELECT * FROM orders WHERE created_at > '2025-01-01';
```

## Monitoring Index Usage

```sql
-- Find unused indexes (MySQL 8.0+)
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'myapp'
ORDER BY object_name, index_name;
```

## Summary

A MySQL index is a sorted B-tree structure that enables fast row lookups without full table scans. Create indexes on columns used in WHERE, JOIN, and ORDER BY clauses with high cardinality. Use composite indexes when queries filter on multiple columns, placing the most selective column first. Always verify index usage with EXPLAIN and periodically audit for unused indexes that add write overhead without benefit.
