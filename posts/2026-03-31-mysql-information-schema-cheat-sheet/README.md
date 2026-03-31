# MySQL INFORMATION_SCHEMA Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Information Schema, Metadata, Cheat Sheet

Description: Quick reference for MySQL INFORMATION_SCHEMA views covering tables, columns, indexes, constraints, routines, and privileges with practical query examples.

---

## Tables and Databases

```sql
-- List all databases
SELECT schema_name FROM information_schema.schemata;

-- List all tables in a database
SELECT table_name, table_type, engine,
       table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;

-- Find largest tables (estimated)
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
ORDER BY size_mb DESC
LIMIT 20;
```

## Columns

```sql
-- Columns for a specific table
SELECT column_name, column_type, is_nullable,
       column_default, extra, column_key
FROM information_schema.columns
WHERE table_schema = 'mydb'
  AND table_name   = 'orders'
ORDER BY ordinal_position;

-- Find all tables with a specific column
SELECT table_schema, table_name
FROM information_schema.columns
WHERE column_name = 'deleted_at';
```

## Indexes

```sql
SELECT table_name, index_name, non_unique,
       seq_in_index, column_name, index_type
FROM information_schema.statistics
WHERE table_schema = 'mydb'
ORDER BY table_name, index_name, seq_in_index;
```

## Constraints

```sql
-- All constraints
SELECT constraint_name, constraint_type, table_name
FROM information_schema.table_constraints
WHERE table_schema = 'mydb';

-- Foreign key details
SELECT kcu.table_name,
       kcu.column_name,
       kcu.referenced_table_name,
       kcu.referenced_column_name
FROM information_schema.key_column_usage kcu
JOIN information_schema.table_constraints tc
  ON tc.constraint_name = kcu.constraint_name
 AND tc.table_schema    = kcu.table_schema
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND kcu.table_schema   = 'mydb';
```

## Routines (Stored Procedures and Functions)

```sql
SELECT routine_name, routine_type, created, last_altered
FROM information_schema.routines
WHERE routine_schema = 'mydb';
```

## Views

```sql
SELECT table_name, view_definition
FROM information_schema.views
WHERE table_schema = 'mydb';
```

## Triggers

```sql
SELECT trigger_name, event_manipulation,
       event_object_table, action_timing
FROM information_schema.triggers
WHERE trigger_schema = 'mydb';
```

## Character Sets and Collations

```sql
-- Available character sets
SELECT character_set_name, default_collate_name, maxlen
FROM information_schema.character_sets
ORDER BY character_set_name;

-- Collations for a character set
SELECT collation_name, is_default
FROM information_schema.collations
WHERE character_set_name = 'utf8mb4';
```

## Privileges

```sql
-- User privileges
SELECT grantee, privilege_type, is_grantable
FROM information_schema.user_privileges;

-- Table privileges
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.table_privileges
WHERE table_schema = 'mydb';
```

## Partitions

```sql
SELECT table_name, partition_name,
       partition_ordinal_position,
       partition_expression,
       table_rows
FROM information_schema.partitions
WHERE table_schema = 'mydb'
  AND partition_name IS NOT NULL
ORDER BY table_name, partition_ordinal_position;
```

## Process List

```sql
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

## Summary

INFORMATION_SCHEMA is the read-only metadata catalog for MySQL. Key views include TABLES (sizes, engines), COLUMNS (schema inspection), STATISTICS (index analysis), KEY_COLUMN_USAGE (foreign key mapping), and ROUTINES (procedure inventory). It is indispensable for database introspection, schema migration tooling, and automated auditing without needing direct access to system tables.
