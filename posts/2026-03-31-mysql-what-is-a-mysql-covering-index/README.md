# What Is a MySQL Covering Index

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Covering Index, Query Optimization

Description: Learn what a MySQL covering index is, how it eliminates secondary lookups by satisfying a query entirely from the index, and how to design covering indexes.

---

A covering index is an index that contains all the columns a query needs, allowing MySQL to return the result directly from the index without accessing the main table (clustered index). This eliminates the "double lookup" that normally occurs with secondary indexes and can dramatically improve read performance.

## The Double Lookup Problem

When MySQL uses a secondary index, it normally performs two steps:
1. Traverse the secondary index B-tree to find matching rows and their primary keys
2. For each primary key, traverse the clustered index to fetch the full row

```sql
CREATE INDEX idx_customer ON orders (customer_id);

EXPLAIN SELECT id, customer_id, total, status FROM orders WHERE customer_id = 42\G
-- key: idx_customer
-- Extra: (nothing about "Using index") -- full row fetch required
```

The secondary index has `customer_id` and the PK `id`, but not `total` or `status`. MySQL must follow each PK back to the clustered index.

## What Makes a Covering Index

A covering index includes all columns referenced in the query's `SELECT`, `WHERE`, `JOIN ON`, and `ORDER BY` clauses.

```sql
-- Create a covering index for this specific query pattern
CREATE INDEX idx_covering ON orders (customer_id, total, status);

EXPLAIN SELECT id, customer_id, total, status FROM orders WHERE customer_id = 42\G
-- key: idx_covering
-- Extra: Using index  <-- This means covering index is used!
```

The phrase "Using index" in the EXPLAIN Extra column confirms MySQL reads only the index, not the table.

## Designing Effective Covering Indexes

The covering index must include:
- Equality filter columns first (WHERE col = ...)
- Range filter columns next (WHERE col > ...)
- Then any remaining SELECT columns not already in the index

```sql
-- Query: filter by customer_id and status, return id, created_at
SELECT id, created_at
FROM orders
WHERE customer_id = 42 AND status = 'paid';

-- Covering index design:
-- 1. Equality filters: customer_id, status
-- 2. SELECT columns not in index: created_at (id comes from PK, always present)
CREATE INDEX idx_cov_orders ON orders (customer_id, status, created_at);

EXPLAIN SELECT id, created_at
FROM orders
WHERE customer_id = 42 AND status = 'paid'\G
-- Extra: Using index
```

## InnoDB Primary Key in Every Index

InnoDB automatically appends the primary key to every secondary index leaf node. This means you never need to explicitly include the PK in a covering index definition.

```sql
-- The PK (id) is implicitly included in all secondary indexes
CREATE INDEX idx_customer ON orders (customer_id);
-- Leaf node contains: (customer_id, id) even though you only specified customer_id

-- This query is covered by idx_customer (returns customer_id and id)
SELECT id, customer_id FROM orders WHERE customer_id = 42;
-- Extra: Using index
```

## Covering Index for COUNT Queries

```sql
-- Add index just to cover COUNT queries without hitting the table
CREATE INDEX idx_status ON orders (status);

SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Extra: Using index (counts from index, no table access)
```

## Trade-offs

Covering indexes are wider (more columns) and therefore larger. They use more disk space and more buffer pool memory. They also add more overhead to INSERT and UPDATE operations since more index entries must be maintained.

```sql
-- Check index sizes to weigh the storage cost
SELECT INDEX_NAME,
  ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = DATABASE()
  AND table_name = 'orders'
  AND stat_name = 'size';
```

## Summary

A covering index satisfies a query entirely from the index without accessing the main table, shown as "Using index" in EXPLAIN. Design covering indexes by including equality filter columns, range columns, then additional SELECT columns in that order. InnoDB automatically includes the primary key in all secondary indexes. Covering indexes trade increased index size and write overhead for significantly faster read performance on frequently executed query patterns.
