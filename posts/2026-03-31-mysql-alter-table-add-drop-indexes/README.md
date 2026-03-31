# How to Add and Drop Indexes with ALTER TABLE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, DDL, Index, ALTER TABLE, Performance, Schema

Description: Add regular, unique, full-text, and composite indexes to MySQL tables with ALTER TABLE and DROP unused indexes to optimise query performance.

---

## How It Works

Indexes are data structures (B-trees by default in InnoDB) that allow MySQL to locate rows without scanning every row in the table. Adding an index speeds up SELECT queries at the cost of slower writes and additional disk space. Dropping an index removes the structure and its overhead.

```mermaid
flowchart LR
    A[SELECT with WHERE col = ?] --> B{Index on col?}
    B -- Yes --> C[Index lookup\nO(log n)]
    B -- No --> D[Full table scan\nO(n)]
    C --> E[Return rows]
    D --> E
```

## Adding Indexes

### Syntax

```sql
ALTER TABLE table_name
    ADD [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name (col1 [ASC|DESC], col2, ...)
    [ALGORITHM=INPLACE] [LOCK=NONE];
```

You can also use `ADD KEY` - it is a synonym for `ADD INDEX`.

### Regular Index

Speeds up lookups on a column that is not unique.

```sql
CREATE TABLE orders (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id    INT UNSIGNED NOT NULL,
    status     VARCHAR(20)  NOT NULL DEFAULT 'pending',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total      DECIMAL(10,2) NOT NULL
);

ALTER TABLE orders ADD INDEX idx_user_id (user_id);
```

### Unique Index

Enforces uniqueness and speeds up lookups.

```sql
ALTER TABLE users ADD UNIQUE INDEX uq_email (email);
```

### Composite Index

An index on multiple columns. The order matters - MySQL can use the index for queries filtering on a prefix of the columns.

```sql
-- Good for queries like: WHERE status = 'pending' ORDER BY created_at
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);

-- Good for: WHERE user_id = ? AND status = ?
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
```

### Full-Text Index

Enables full-text search with `MATCH ... AGAINST`.

```sql
CREATE TABLE articles (
    id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body  TEXT         NOT NULL
);

ALTER TABLE articles ADD FULLTEXT INDEX ft_title_body (title, body);

-- Use the full-text index
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('mysql indexing' IN BOOLEAN MODE);
```

### Prefix Index (on TEXT/BLOB or long VARCHAR)

Index only the first N characters to save space.

```sql
ALTER TABLE products ADD INDEX idx_description_prefix (description(100));
```

## Dropping Indexes

### Syntax

```sql
ALTER TABLE table_name DROP INDEX index_name;
```

For primary keys:

```sql
ALTER TABLE table_name DROP PRIMARY KEY;
```

### Example

```sql
ALTER TABLE orders DROP INDEX idx_user_id;
ALTER TABLE users  DROP INDEX uq_email;
```

## Viewing Existing Indexes

```sql
SHOW INDEX FROM orders;
```

```text
+--------+-------+--------------------+--------------+-------------+-------------+
| Table  | Non_unique | Key_name      | Seq_in_index | Column_name | Index_type  |
+--------+-------+--------------------+--------------+-------------+-------------+
| orders |      0 | PRIMARY           |            1 | id          | BTREE       |
| orders |      1 | idx_user_id       |            1 | user_id     | BTREE       |
| orders |      1 | idx_status_created|            1 | status      | BTREE       |
| orders |      1 | idx_status_created|            2 | created_at  | BTREE       |
+--------+-------+--------------------+--------------+-------------+-------------+
```

Use `information_schema` for programmatic access.

```sql
SELECT INDEX_NAME, COLUMN_NAME, NON_UNIQUE, SEQ_IN_INDEX
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

## Combining Multiple Changes in One Statement

```sql
ALTER TABLE orders
    ADD INDEX idx_status_created (status, created_at),
    ADD INDEX idx_user_status    (user_id, status),
    DROP INDEX idx_old_index,
    ALGORITHM=INPLACE,
    LOCK=NONE;
```

## Finding Unused Indexes

Query `performance_schema` to identify indexes that have never been used.

```sql
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = DATABASE()
  AND index_name != 'PRIMARY'
  AND count_star = 0
ORDER BY object_name, index_name;
```

Remove unused indexes to improve write performance and reduce storage.

## Finding Missing Indexes

```sql
-- Check for full table scans in the slow query log summary
SELECT digest_text, count_star, sum_rows_examined, sum_rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE sum_rows_examined / NULLIF(sum_rows_sent, 0) > 100
ORDER BY sum_rows_examined DESC
LIMIT 20;
```

## Best Practices

- Index every column used in `WHERE`, `JOIN ON`, and `ORDER BY` clauses that have high cardinality.
- Use composite indexes instead of multiple single-column indexes for queries that filter on multiple columns.
- Put the most selective column first in a composite index.
- Use `ALGORITHM=INPLACE, LOCK=NONE` for online index operations on large production tables.
- Remove duplicate indexes (e.g., `(a)` and `(a, b)` - the second covers the first).
- Monitor index usage with `performance_schema` and drop indexes that are never used.

## Summary

`ALTER TABLE ADD INDEX` creates B-tree indexes that dramatically speed up SELECT queries. Choose regular indexes for non-unique columns, unique indexes for natural keys, composite indexes for multi-column filters, and full-text indexes for text search. `ALTER TABLE DROP INDEX` removes unused indexes. Use `SHOW INDEX FROM table` and `information_schema.STATISTICS` to audit existing indexes, and `performance_schema.table_io_waits_summary_by_index_usage` to identify unused ones.
