# What Is a Descending Index in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Descending Index, Index

Description: Understand what a MySQL descending index is, how it differs from ascending indexes, and when descending indexes eliminate filesort operations for ORDER BY queries.

---

A descending index in MySQL stores index entries in descending order rather than the default ascending order. Introduced as a functional feature in MySQL 8.0 (the syntax existed in 5.7 but was silently ignored), descending indexes optimize queries that sort columns in descending order or use mixed-direction ORDER BY clauses.

## How Standard Indexes Handle Descending Sorts

By default, MySQL B-tree indexes store entries in ascending order. To retrieve rows in descending order, MySQL can traverse the index backward. This works well for single-column descending sorts.

However, for composite ORDER BY with mixed directions (ASC on one column, DESC on another), a single ascending index cannot satisfy the sort order without a `filesort` operation.

```sql
-- Table with ascending composite index
CREATE TABLE events (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  user_id    INT NOT NULL,
  occurred_at DATETIME NOT NULL,
  INDEX idx_user_time (user_id ASC, occurred_at ASC)
) ENGINE=InnoDB;

-- Query with mixed sort direction
EXPLAIN SELECT * FROM events
WHERE user_id IN (1, 2, 3)
ORDER BY user_id ASC, occurred_at DESC\G
-- Extra: Using filesort (index cannot serve mixed direction without DESC index)
```

## Creating a Descending Index

```sql
-- Single-column descending index
CREATE INDEX idx_occurred_desc ON events (occurred_at DESC);

-- Composite index with mixed direction (MySQL 8.0+)
CREATE INDEX idx_user_time_mixed ON events (user_id ASC, occurred_at DESC);

-- Now the mixed-direction query uses the index
EXPLAIN SELECT * FROM events
WHERE user_id IN (1, 2, 3)
ORDER BY user_id ASC, occurred_at DESC\G
-- Extra: (no filesort)
```

## MySQL 5.7 vs 8.0 Behavior

In MySQL 5.7, `DESC` in an index definition was accepted but ignored - all indexes were effectively ascending. In MySQL 8.0, descending indexes are truly stored in descending order.

```sql
-- This behaves differently in 5.7 vs 8.0
CREATE INDEX idx_time_desc ON events (occurred_at DESC);

-- MySQL 5.7: actually stored ASC, DESC keyword ignored
-- MySQL 8.0: stored DESC, optimizer uses it for DESC sorts
```

## When Descending Indexes Help

Descending indexes are most valuable in two scenarios:

**Latest records queries**: When your application frequently queries for the most recent N records.

```sql
-- With idx_occurred_desc, this avoids filesort on large tables
SELECT * FROM events ORDER BY occurred_at DESC LIMIT 20;
-- Extra: Using index (or Backward index scan without filesort)
```

**Mixed-direction ORDER BY**: When sorting requires ascending on one column and descending on another - the most common use case for descending indexes.

```sql
-- Common pattern: sort by category ASC, price DESC
CREATE INDEX idx_cat_price ON products (category_id ASC, price DESC);

SELECT * FROM products
WHERE category_id = 5
ORDER BY category_id ASC, price DESC
LIMIT 10;
-- Extra: Using index condition (no filesort)
```

## Checking EXPLAIN for Backward Index Scan

```sql
EXPLAIN SELECT * FROM events ORDER BY occurred_at DESC LIMIT 10\G
-- Extra: Backward index scan; Using index
```

"Backward index scan" means MySQL traverses the ascending index in reverse - this is efficient but slightly slower than a forward scan. A true descending index avoids the backward scan.

## Single Column Descending - Is It Worth It?

For queries sorting a single column descending, MySQL can efficiently traverse an ascending index backward. Creating a separate descending index for a single column is rarely necessary.

```sql
-- Ascending index traversed backward is usually fine
CREATE INDEX idx_time ON events (occurred_at);  -- ASC (default)
SELECT * FROM events ORDER BY occurred_at DESC LIMIT 100;
-- Extra: Backward index scan (efficient enough)
```

Reserve true descending indexes for mixed-direction composite ORDER BY cases.

## Summary

Descending indexes in MySQL 8.0 store B-tree entries in descending order, eliminating filesort operations for queries that sort columns in descending order or use composite ORDER BY with mixed directions (ASC on one column, DESC on another). For single-column descending sorts, MySQL can use backward index scan on an ascending index efficiently. Descending indexes are most valuable for composite indexes with mixed sort directions.
