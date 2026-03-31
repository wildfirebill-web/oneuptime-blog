# How to Use CREATE INDEX Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Indexing, Performance, Database Optimization

Description: Learn how to use the CREATE INDEX statement in MySQL to speed up query performance by creating single-column, composite, and unique indexes.

---

## What Is CREATE INDEX in MySQL

The `CREATE INDEX` statement allows you to add an index to an existing table in MySQL. Indexes improve the speed of data retrieval by reducing the number of rows MySQL must scan during queries. Without indexes, MySQL performs a full table scan for every query, which becomes expensive as tables grow.

MySQL supports several types of indexes: regular (BTree), UNIQUE, FULLTEXT, and SPATIAL.

## Basic Syntax

```sql
CREATE [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name
    [index_type]
    ON table_name (key_part, ...)
    [index_option];
```

The most common form for a single-column index:

```sql
CREATE INDEX idx_email ON users (email);
```

## Creating a Single-Column Index

```sql
-- Create a table
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    status VARCHAR(20),
    created_at DATETIME
);

-- Add an index on customer_id to speed up lookups
CREATE INDEX idx_customer_id ON orders (customer_id);
```

After creating this index, queries that filter by `customer_id` will use it automatically:

```sql
SELECT * FROM orders WHERE customer_id = 42;
```

## Creating a Composite Index

A composite index covers multiple columns and helps queries that filter or sort by those columns together:

```sql
CREATE INDEX idx_customer_status ON orders (customer_id, status);
```

This index benefits queries like:

```sql
SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

The leftmost prefix rule applies - this index also helps queries filtering only by `customer_id`, but not queries filtering only by `status`.

## Creating a UNIQUE Index

A UNIQUE index enforces uniqueness across the indexed column(s):

```sql
CREATE UNIQUE INDEX idx_unique_email ON users (email);
```

Attempting to insert a duplicate email will result in an error:

```sql
INSERT INTO users (email) VALUES ('test@example.com'); -- OK
INSERT INTO users (email) VALUES ('test@example.com'); -- ERROR: Duplicate entry
```

## Creating a FULLTEXT Index

FULLTEXT indexes enable full-text search on `CHAR`, `VARCHAR`, and `TEXT` columns:

```sql
CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    body TEXT
);

CREATE FULLTEXT INDEX idx_fulltext_body ON articles (body);
```

Use FULLTEXT indexes with `MATCH ... AGAINST`:

```sql
SELECT * FROM articles
WHERE MATCH(body) AGAINST('MySQL performance' IN NATURAL LANGUAGE MODE);
```

## Index Options: Key Length and Sort Order

For string columns you can specify a prefix length to index only part of the value:

```sql
CREATE INDEX idx_name_prefix ON users (last_name(10));
```

You can also specify sort direction for composite indexes (useful for `ORDER BY` optimization):

```sql
CREATE INDEX idx_created_desc ON orders (created_at DESC);
```

## Checking If an Index Was Created

Use `SHOW INDEX` or `SHOW CREATE TABLE` to verify:

```sql
SHOW INDEX FROM orders;
```

```sql
SHOW CREATE TABLE orders\G
```

You can also query the information schema:

```sql
SELECT index_name, column_name, non_unique
FROM information_schema.statistics
WHERE table_schema = 'mydb'
  AND table_name = 'orders';
```

## Using EXPLAIN to Verify Index Usage

After creating an index, confirm MySQL uses it with `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

Look for your index name in the `key` column. If `key` is `NULL`, MySQL is not using the index.

## When NOT to Create Indexes

Indexes have trade-offs. Avoid creating indexes when:

- The table is small (full scans are cheaper)
- The column has very low cardinality (e.g., a boolean column)
- The table receives heavy write traffic and index maintenance overhead is costly
- The column is rarely used in `WHERE`, `JOIN`, or `ORDER BY` clauses

## Summary

The `CREATE INDEX` statement is essential for MySQL query optimization. Use single-column indexes for frequently filtered columns, composite indexes for multi-column filter or sort patterns, and UNIQUE indexes to enforce data integrity. Always verify index usage with `EXPLAIN` after creation to ensure your queries benefit from the new index.
