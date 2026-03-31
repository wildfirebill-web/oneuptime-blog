# How to Create a Composite (Multi-Column) Index in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Composite Index, Query Optimization, Performance

Description: Learn how to create composite multi-column indexes in MySQL, understand column order importance, and use them to optimize multi-condition queries.

---

## Overview

A composite index (also called a multi-column index) is an index that spans two or more columns. It is significantly more powerful than multiple single-column indexes for queries that filter on multiple columns simultaneously.

The key rule of composite indexes is the **leftmost prefix rule**: MySQL can use the index for queries that reference the leading columns of the index, but cannot skip over columns in the middle.

## Basic Syntax

```sql
CREATE INDEX index_name ON table_name (column1, column2, column3);
```

## Creating a Composite Index

```sql
CREATE TABLE orders (
  id INT NOT NULL AUTO_INCREMENT,
  user_id INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  total DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (id)
);

-- Composite index on user_id and status
CREATE INDEX idx_user_status ON orders (user_id, status);
```

## The Leftmost Prefix Rule

With a composite index on `(user_id, status)`, MySQL can use the index for:

```sql
-- Uses idx_user_status (both columns)
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';

-- Uses idx_user_status (leftmost prefix only)
SELECT * FROM orders WHERE user_id = 42;

-- Does NOT use idx_user_status (skips user_id)
SELECT * FROM orders WHERE status = 'pending';
```

The third query requires a separate index on `status` alone if you want index coverage.

## Column Order Matters

Put the most selective column first, and consider your most common query patterns:

```sql
-- If most queries filter by user_id, put it first
CREATE INDEX idx_user_status ON orders (user_id, status);

-- If most queries filter by status across all users, separate index may be better
CREATE INDEX idx_status ON orders (status);
```

## Three-Column Composite Index

```sql
CREATE INDEX idx_user_status_date ON orders (user_id, status, created_at);
```

This index supports all of these query patterns:

```sql
-- Uses all three columns
SELECT * FROM orders
WHERE user_id = 42 AND status = 'paid' AND created_at >= '2025-01-01';

-- Uses first two columns
SELECT * FROM orders WHERE user_id = 42 AND status = 'paid';

-- Uses first column only
SELECT * FROM orders WHERE user_id = 42;

-- Does NOT use the index effectively
SELECT * FROM orders WHERE status = 'paid' AND created_at >= '2025-01-01';
```

## Using EXPLAIN to Verify Usage

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'pending'\G
```

```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: orders
         type: ref
possible_keys: idx_user_status
          key: idx_user_status
      key_len: 86
          ref: const,const
         rows: 3
        Extra: NULL
```

The `key` showing `idx_user_status` and `ref: const,const` confirms both columns are used.

## Composite Index vs Multiple Single-Column Indexes

```sql
-- Two separate indexes
CREATE INDEX idx_user ON orders (user_id);
CREATE INDEX idx_status ON orders (status);

-- One composite index
CREATE INDEX idx_user_status ON orders (user_id, status);
```

For a query with `WHERE user_id = 42 AND status = 'paid'`, the composite index is more efficient. MySQL can only use one index per table in most cases, and the composite index filters on both conditions in a single lookup.

## Adding a Composite Index to an Existing Table

```sql
ALTER TABLE orders ADD INDEX idx_user_status_date (user_id, status, created_at);
```

## Viewing Composite Index Columns

```sql
SHOW INDEX FROM orders;
```

```text
+--------+------------+--------------------+--------------+-------------+
| Table  | Non_unique | Key_name           | Seq_in_index | Column_name |
+--------+------------+--------------------+--------------+-------------+
| orders |          0 | PRIMARY            |            1 | id          |
| orders |          1 | idx_user_status    |            1 | user_id     |
| orders |          1 | idx_user_status    |            2 | status      |
+--------+------------+--------------------+--------------+-------------+
```

`Seq_in_index` shows the position of each column in the composite index.

## Summary

Composite indexes are a powerful tool for optimizing multi-condition queries in MySQL. The leftmost prefix rule determines which queries benefit from the index, so column order is critical - put the most commonly filtered column first. Use `EXPLAIN` to verify MySQL is using your composite index and check `key_len` to see how many columns of the index are being used in a given query.
