# How to Drop an Index in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, DDL, InnoDB, Schema

Description: Learn how to drop indexes in MySQL using DROP INDEX and ALTER TABLE, how to safely remove unused indexes, and what to check before deleting an index.

---

## Two Ways to Drop an Index

MySQL provides two equivalent syntaxes for removing an index:

```sql
-- Syntax 1: DROP INDEX
DROP INDEX index_name ON table_name;

-- Syntax 2: ALTER TABLE
ALTER TABLE table_name DROP INDEX index_name;
```

`ALTER TABLE` is preferred when you are combining the index removal with other schema changes in a single statement.

## Dropping a Regular Index

```sql
DROP INDEX idx_status ON orders;
-- equivalent to:
ALTER TABLE orders DROP INDEX idx_status;
```

## Dropping a Unique Index

```sql
DROP INDEX uk_email ON users;
ALTER TABLE users DROP INDEX uk_email;
```

Dropping a unique index removes the uniqueness constraint as well - duplicate values will be permitted after the drop.

## Dropping a Composite Index

```sql
ALTER TABLE order_items DROP INDEX uk_order_product;
```

## Dropping a Primary Key

Dropping a primary key requires `ALTER TABLE` with `DROP PRIMARY KEY`:

```sql
ALTER TABLE legacy_table DROP PRIMARY KEY;
```

Be cautious - dropping a primary key on an InnoDB table with foreign keys referencing it is not permitted until the foreign keys are dropped first.

## Combining with Other Alterations

```sql
ALTER TABLE orders
    DROP INDEX idx_legacy_flag,
    DROP INDEX idx_old_status,
    ADD INDEX idx_status_created (status, created_at),
    ALGORITHM = INPLACE,
    LOCK = NONE;
```

Combining drops and additions in one statement reduces the number of table rebuilds.

## Checking if an Index Exists Before Dropping

MySQL does not support `DROP INDEX IF EXISTS` natively. Use a stored procedure or a conditional shell command:

```sql
-- Check first
SELECT COUNT(*) FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders'
  AND INDEX_NAME = 'idx_status';

-- Then drop if result is > 0
```

Or use a prepared statement in application code to check before executing the drop.

## Dropping Indexes Online (INPLACE)

For large tables, drop indexes online to avoid blocking writes:

```sql
ALTER TABLE orders
    DROP INDEX idx_status,
    ALGORITHM = INPLACE,
    LOCK = NONE;
```

## Before Dropping: Validate Usage

Before removing an index, confirm it is not being used:

```sql
SELECT OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME,
       COUNT_READ, COUNT_FETCH
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'your_database'
  AND OBJECT_NAME = 'orders'
  AND INDEX_NAME = 'idx_status';
```

If `COUNT_READ` and `COUNT_FETCH` are zero since the server started, the index has not been used.

## Summary

Use `DROP INDEX` or `ALTER TABLE ... DROP INDEX` to remove indexes in MySQL. Always verify index usage through `performance_schema` before dropping, consider making the index invisible first as a safer intermediate step, and use `ALGORITHM=INPLACE, LOCK=NONE` when dropping indexes on large production tables to avoid write blocking.
