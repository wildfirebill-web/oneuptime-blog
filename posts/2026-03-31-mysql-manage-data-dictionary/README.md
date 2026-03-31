# How to Manage the MySQL Data Dictionary

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Dictionary, Administration, Metadata, InnoDB

Description: Learn how MySQL 8.0 manages the data dictionary, how to query metadata, and how the transactional data dictionary differs from previous versions.

---

In MySQL 8.0, the data dictionary was completely redesigned. Instead of storing metadata in `.frm` files and non-transactional MyISAM tables, MySQL 8.0 stores all metadata in InnoDB tables within the `mysql` schema. This change makes data dictionary operations transactional, crash-safe, and consistent.

## What Changed in MySQL 8.0

Prior to MySQL 8.0:
- Table definitions stored in `.frm` files
- Grant tables in MyISAM format
- Metadata not covered by ACID guarantees

In MySQL 8.0:
- All metadata stored in the `mysql.ibd` InnoDB tablespace
- Transactional and crash-safe
- Dictionary views exposed through `information_schema` and `performance_schema`

## Querying the Data Dictionary

The primary interface to the data dictionary is `information_schema`. The views are now backed by InnoDB tables, making them significantly faster in MySQL 8.0:

```sql
-- List all tables in a database
SELECT TABLE_NAME, TABLE_TYPE, ENGINE, ROW_FORMAT, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY TABLE_NAME;
```

```sql
-- List all columns in a table
SELECT COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE, COLUMN_DEFAULT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;
```

```sql
-- List all indexes
SELECT INDEX_NAME, NON_UNIQUE, COLUMN_NAME, SEQ_IN_INDEX
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

## Using InnoDB-Specific Dictionary Tables

MySQL 8.0 exposes InnoDB internal metadata through `information_schema`:

```sql
-- View all InnoDB tables with their tablespace info
SELECT NAME, SPACE, ROW_FORMAT, SPACE_TYPE
FROM information_schema.INNODB_TABLES
WHERE NAME LIKE 'mydb/%'
ORDER BY NAME;
```

```sql
-- Check indexes stored in the data dictionary
SELECT NAME, TABLE_ID, PAGE_NO, TYPE
FROM information_schema.INNODB_INDEXES
WHERE NAME != 'PRIMARY'
LIMIT 20;
```

## Viewing Stored Programs in the Dictionary

```sql
-- List stored procedures and functions
SELECT ROUTINE_NAME, ROUTINE_TYPE, CREATED, LAST_ALTERED
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = 'mydb';
```

```sql
-- List triggers
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE, ACTION_TIMING
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb';
```

## Data Dictionary and mysqldump

When exporting the data dictionary structure (table schemas) only, without row data:

```bash
mysqldump -u root -p --no-data mydb > mydb_schema.sql
```

The exported file contains `CREATE TABLE` and `CREATE VIEW` statements derived from the data dictionary.

## Checking for Orphaned Files

MySQL 8.0 tracks all tablespace files in the data dictionary. If a `.ibd` file exists but is not in the dictionary, MySQL reports it as an orphan. Check for these:

```sql
SELECT FILE_NAME
FROM information_schema.FILES
WHERE FILE_TYPE = 'TABLESPACE'
  AND FILE_NAME NOT LIKE 'innodb_%'
ORDER BY FILE_NAME;
```

Compare with actual `.ibd` files on disk to identify orphans left from failed operations.

## Upgrading and the Data Dictionary

When upgrading from MySQL 5.7 to 8.0, the upgrade process converts the old metadata to the new dictionary format. Use `mysqlcheck` to verify tables are compatible before upgrading:

```bash
mysqlcheck -u root -p --check-upgrade --all-databases
```

## Summary

MySQL 8.0's transactional data dictionary improves reliability by storing all metadata in InnoDB, eliminating `.frm` files and making metadata operations crash-safe. Access it through `information_schema` views for table definitions, column details, and index metadata. Use InnoDB-specific views for internal storage details, and rely on `mysqldump --no-data` to export the dictionary-derived schema for documentation or migration purposes.
