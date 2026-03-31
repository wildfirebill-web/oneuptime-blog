# How to Copy a Table with Data in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Table, Data Migration, Database, Insert

Description: Learn how to copy a table along with its data in MySQL using CREATE TABLE SELECT, INSERT INTO SELECT, and mysqldump.

---

## Overview

Copying a table with all its data is a frequent task in MySQL for creating backups, migrating data, or setting up test environments. MySQL offers several approaches, each with different tradeoffs.

## Using CREATE TABLE AS SELECT

The most direct method combines table creation and data insertion in one statement:

```sql
CREATE TABLE employees_backup AS
SELECT * FROM employees;
```

This creates `employees_backup` with all rows from `employees`. However, this approach does NOT copy indexes or constraints. Only column names and data types are preserved.

## Preserving Indexes: Two-Step Approach

To copy both the structure (with indexes) and the data, use two statements:

```sql
-- Step 1: Copy structure including indexes
CREATE TABLE employees_backup LIKE employees;

-- Step 2: Copy the data
INSERT INTO employees_backup SELECT * FROM employees;
```

This is the most reliable method for a full table copy in MySQL.

## Copying a Subset of Data

You can copy only specific rows using a WHERE clause:

```sql
CREATE TABLE employees_2024 LIKE employees;

INSERT INTO employees_2024
SELECT * FROM employees
WHERE YEAR(created_at) = 2024;
```

## Copying Selected Columns

To copy only specific columns:

```sql
CREATE TABLE employees_names AS
SELECT id, first_name, last_name FROM employees;
```

## Copying Across Databases

You can copy a table to a different database:

```sql
CREATE TABLE archive_db.employees LIKE production_db.employees;

INSERT INTO archive_db.employees
SELECT * FROM production_db.employees;
```

This is useful when separating archival data from the production schema.

## Using mysqldump for Large Tables

For large tables, `mysqldump` is more efficient as it handles data in chunks and allows compression:

```bash
mysqldump -u root -p production_db employees > employees_dump.sql
mysql -u root -p archive_db < employees_dump.sql
```

Or copy directly between databases on the same server:

```bash
mysqldump -u root -p production_db employees | mysql -u root -p archive_db
```

## Checking Row Counts After Copy

After copying, verify the row counts match:

```sql
SELECT COUNT(*) FROM employees;
SELECT COUNT(*) FROM employees_backup;
```

## Handling AUTO_INCREMENT

When using `CREATE TABLE ... LIKE`, the `AUTO_INCREMENT` counter resets to 1. After inserting data, reset the counter to the correct value:

```sql
-- Find the max id
SELECT MAX(id) FROM employees_backup;

-- Set AUTO_INCREMENT accordingly
ALTER TABLE employees_backup AUTO_INCREMENT = 10001;
```

## Monitoring Copy Operations with OneUptime

Large table copies can generate significant I/O load. Use OneUptime to monitor your MySQL host metrics (CPU, disk I/O, replication lag) during copy operations to ensure the primary database remains healthy.

## Summary

Use `CREATE TABLE ... LIKE` followed by `INSERT INTO ... SELECT` to copy both structure and data in MySQL while preserving indexes. For large tables, prefer `mysqldump` for efficiency. Always verify row counts after the copy to confirm data integrity.
