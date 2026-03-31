# How to Fix 'Duplicate column name' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Column, Error, Schema, SQL

Description: Resolve 'Duplicate column name' errors in ClickHouse caused by star expansion in JOINs, UNION ALL, and DDL statements with repeated column names.

---

ClickHouse raises "Duplicate column name" when a query or DDL produces a result set with two or more columns sharing the same name. This frequently occurs with `SELECT *` in JOINs, UNION ALL operations, or CREATE TABLE statements.

## Error in SELECT * with JOINs

When both tables in a JOIN have a column with the same name:

```sql
-- Both 'events' and 'users' have a 'user_id' column:
SELECT * FROM events JOIN users USING (user_id);
-- Error: Duplicate column name 'user_id'
```

Fix by listing columns explicitly:

```sql
SELECT
    e.event_id,
    e.event_type,
    e.event_time,
    u.user_name,
    u.email
FROM events e
JOIN users u ON e.user_id = u.user_id;
```

Or use aliases:

```sql
SELECT
    e.*,
    u.user_name,
    u.email
FROM events e
JOIN users u ON e.user_id = u.user_id;
```

## Error in CREATE TABLE

```sql
-- Duplicate column in DDL:
CREATE TABLE my_table (
    user_id   UInt64,
    event_type String,
    user_id   String  -- Duplicate!
) ENGINE = MergeTree() ORDER BY user_id;
```

Fix by renaming the duplicate:

```sql
CREATE TABLE my_table (
    user_id    UInt64,
    event_type String,
    source_id  String
) ENGINE = MergeTree() ORDER BY user_id;
```

## Error in UNION ALL

```sql
-- Column names collide after UNION ALL:
SELECT user_id, count() AS user_id FROM events GROUP BY user_id;
-- Error: Duplicate column name 'user_id'

-- Fix: use distinct aliases
SELECT user_id, count() AS event_count FROM events GROUP BY user_id;
```

## Error After ADD COLUMN

```sql
-- Adding a column that already exists:
ALTER TABLE my_table ADD COLUMN event_type String;
-- Error: Duplicate column name 'event_type'

-- Check existing columns first:
SELECT name FROM system.columns
WHERE table = 'my_table' AND name = 'event_type';

-- Use IF NOT EXISTS (ClickHouse 22.6+):
ALTER TABLE my_table ADD COLUMN IF NOT EXISTS event_type String;
```

## Fix Star Expansion Issues

Disable star expansion ambiguity with explicit columns or change your asterisk policy:

```sql
SET asterisk_include_alias_columns = 0;

-- Or always list columns explicitly in production queries
SELECT col1, col2, col3
FROM (
    SELECT t1.col1, t2.col1 AS col2, t1.col3
    FROM t1 JOIN t2 ON t1.id = t2.id
);
```

## Summary

"Duplicate column name" in ClickHouse is caused by `SELECT *` on JOINs with shared column names, UNION ALL branches with conflicting aliases, or DDL with repeated column names. The fix is always to list columns explicitly, use table-qualified names in joins, and add `IF NOT EXISTS` guards to `ALTER TABLE ADD COLUMN` statements.
