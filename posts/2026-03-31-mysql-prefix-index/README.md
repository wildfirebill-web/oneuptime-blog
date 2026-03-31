# How to Create a Prefix Index in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, String, Performance, InnoDB

Description: Learn how to create prefix indexes in MySQL to index only the first N characters of TEXT or VARCHAR columns, reducing index size while retaining query performance.

---

## What Is a Prefix Index?

A prefix index indexes only the first N characters of a `CHAR`, `VARCHAR`, `TEXT`, or `BLOB` column instead of the full value. This reduces the size of the index on disk and in the InnoDB buffer pool while still accelerating `WHERE` and `JOIN` queries on that column.

Prefix indexes are required for `TEXT` and `BLOB` columns - MySQL does not allow full-column indexes on them.

## Syntax

```sql
CREATE INDEX index_name ON table_name (column_name(prefix_length));
```

## Creating a Prefix Index

```sql
-- Index only the first 10 characters of the email column
CREATE INDEX idx_email_prefix ON users(email(10));

-- Using ALTER TABLE
ALTER TABLE users ADD INDEX idx_email_prefix (email(10));
```

## Required for TEXT and BLOB Columns

```sql
CREATE TABLE articles (
    id      INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title   VARCHAR(500) NOT NULL,
    content LONGTEXT NOT NULL,
    INDEX idx_content (content(50))  -- prefix required for TEXT/BLOB
);
```

## Choosing the Right Prefix Length

The goal is to find the shortest prefix that still provides high selectivity (many distinct values). Use this query to compare prefix lengths:

```sql
SELECT
    COUNT(DISTINCT LEFT(email, 5))  AS sel_5,
    COUNT(DISTINCT LEFT(email, 10)) AS sel_10,
    COUNT(DISTINCT LEFT(email, 20)) AS sel_20,
    COUNT(DISTINCT email)           AS sel_full
FROM users;
```

```text
+---------+----------+----------+----------+
| sel_5   | sel_10   | sel_20   | sel_full |
+---------+----------+----------+----------+
| 91234   | 98871    | 99902    | 100000   |
+---------+----------+----------+----------+
```

If `sel_10` is close to `sel_full`, a prefix of 10 is sufficient.

## Limitations of Prefix Indexes

Prefix indexes cannot be used as covering indexes because MySQL cannot reconstruct the full column value from the index entry alone.

```sql
-- This query CANNOT use the prefix index as a covering index
-- MySQL must access the table row to verify the full email value
SELECT id FROM users WHERE email = 'alice@example.com';
```

Prefix indexes also cannot enforce uniqueness on the full value - a unique prefix index only guarantees prefix uniqueness.

## Prefix Index on a Composite Key

```sql
ALTER TABLE blog_posts
    ADD INDEX idx_author_title (author_id, title(30));
```

Only the `title` portion uses a prefix; `author_id` is indexed in full.

## Verifying the Index in EXPLAIN

```sql
EXPLAIN SELECT id FROM users WHERE email LIKE 'alice%'\G
```

The output should show `key: idx_email_prefix` confirming the prefix index is used.

## Summary

Prefix indexes allow you to index long string columns efficiently by indexing only the first N characters. Use the selectivity query to pick the smallest prefix that retains near-full selectivity. Remember that prefix indexes cannot cover queries or enforce full-value uniqueness, and they are mandatory for indexing `TEXT` and `BLOB` columns in MySQL.
