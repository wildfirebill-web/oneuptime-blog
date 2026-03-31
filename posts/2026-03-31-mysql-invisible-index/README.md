# How to Create an Invisible Index in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Performance, InnoDB, Query

Description: Learn how to use invisible indexes in MySQL 8 to safely test the impact of dropping an index without actually removing it from the table.

---

## What Is an Invisible Index?

An invisible index exists in the table and is maintained by InnoDB on every write, but the query optimizer ignores it entirely. This gives you a safe way to simulate dropping an index and observe query performance effects before committing to the actual removal.

## Creating an Invisible Index

```sql
CREATE INDEX idx_status ON orders(status) INVISIBLE;
```

Or make an existing index invisible:

```sql
ALTER TABLE orders ALTER INDEX idx_status INVISIBLE;
```

## Making an Invisible Index Visible Again

```sql
ALTER TABLE orders ALTER INDEX idx_status VISIBLE;
```

## Viewing Index Visibility

```sql
SHOW INDEX FROM orders\G
```

Look for the `Visible` column:

```text
...
Visible: NO
...
```

Or query the information schema:

```sql
SELECT INDEX_NAME, IS_VISIBLE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders';
```

## The Workflow: Test Before You Drop

The recommended workflow for safely removing an index is:

1. Make the index invisible and monitor query performance and slow query logs.
2. If no regressions appear after a suitable observation period (hours or days), drop the index.
3. If regressions appear, make the index visible again immediately.

```sql
-- Step 1: hide the index
ALTER TABLE orders ALTER INDEX idx_status INVISIBLE;

-- Step 2: observe performance for 24 hours
-- Step 3a: if safe, remove it
ALTER TABLE orders DROP INDEX idx_status;

-- Step 3b: if queries regressed, restore it
ALTER TABLE orders ALTER INDEX idx_status VISIBLE;
```

## Forcing the Optimizer to Use an Invisible Index

For diagnostic purposes, you can instruct the current session to consider invisible indexes:

```sql
SET SESSION optimizer_switch = 'use_invisible_indexes=on';

-- Now this session will consider invisible indexes
EXPLAIN SELECT * FROM orders WHERE status = 'pending'\G
```

This is useful to confirm that the invisible index would be chosen if it were visible.

## Limitations

- The primary key cannot be made invisible.
- Invisible indexes still consume storage and impose write overhead.
- Index visibility is a table-level property, not session-level (unlike `use_invisible_indexes` which is session-level).

## Practical Example

```sql
-- Suppose idx_legacy_flag was added years ago and may be unused
ALTER TABLE users ALTER INDEX idx_legacy_flag INVISIBLE;

-- Check slow query log and monitoring for 48 hours
-- No regressions found - proceed
ALTER TABLE users DROP INDEX idx_legacy_flag;
```

## Summary

Invisible indexes in MySQL 8 are the safest way to evaluate the impact of removing an index. Mark the index invisible, observe for regressions, and only drop it once you have confirmed no queries relied on it. Use `use_invisible_indexes=on` in a session to force the optimizer to consider invisible indexes for diagnostic `EXPLAIN` queries.
