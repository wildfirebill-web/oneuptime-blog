# How to Use InnoDB File-Per-Table Tablespaces in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Tablespace, Storage, Configuration

Description: Learn how InnoDB file-per-table tablespaces work in MySQL, how to enable them, and when to use them for better storage management.

---

The `innodb_file_per_table` setting controls whether InnoDB stores each table's data and indexes in its own `.ibd` file or in the shared system tablespace. File-per-table mode is the default since MySQL 5.6 and is recommended for most workloads.

## Checking and Enabling File-Per-Table Mode

```sql
-- Check current setting
SHOW VARIABLES LIKE 'innodb_file_per_table';

-- Enable file-per-table mode at runtime
SET GLOBAL innodb_file_per_table = ON;
```

Persist the setting in `my.cnf`:

```text
[mysqld]
innodb_file_per_table=ON
```

Changing this variable only affects newly created tables. Existing tables remain in their current tablespace.

## How It Works

When `innodb_file_per_table=ON`, each table gets a dedicated `.ibd` file in the schema directory:

```bash
ls /var/lib/mysql/mydb/
# orders.ibd
# customers.ibd
# products.ibd
```

The `.ibd` file contains the table's data pages, index pages, and metadata. Each file has its own tablespace ID tracked in `information_schema`.

## Benefits of File-Per-Table Mode

The primary advantage is reclaiming disk space. When you `DROP TABLE` or `TRUNCATE TABLE`, MySQL removes the entire `.ibd` file, immediately returning space to the OS. With the shared system tablespace, space is never returned to the OS even after large deletions.

```sql
-- This reclaims disk space immediately in file-per-table mode
DROP TABLE old_logs;

-- OPTIMIZE TABLE reclaims space from deletions within the file
OPTIMIZE TABLE orders;
```

Other benefits include easier individual table backups with tools like `Xtrabackup`, the ability to place specific tables on different storage devices, and per-table encryption.

## Verifying Table Locations

```sql
-- Check the file path for a specific table
SELECT FILE_NAME, TABLESPACE_NAME, FILE_TYPE
FROM information_schema.FILES
WHERE FILE_NAME LIKE '%orders%';

-- Or check the tablespace for a table
SELECT NAME, SPACE_TYPE
FROM information_schema.INNODB_TABLES
WHERE NAME = 'mydb/orders';
```

## Converting Tables Between Modes

To move an existing table from the system tablespace to file-per-table mode, run `ALTER TABLE` with `FORCE` or rebuild it:

```sql
-- Rebuild the table into its own .ibd file
ALTER TABLE orders ENGINE=InnoDB;
```

This performs a full table rebuild, which can be slow. Use `pt-online-schema-change` for large tables to avoid locking.

## Drawbacks to Consider

With many tables, each having its own file increases the OS file descriptor count. Very large numbers of tables (tens of thousands) can create file system overhead. In such cases, consider general tablespaces to group related tables.

```sql
-- Check current open files
SHOW STATUS LIKE 'Innodb_num_open_files';
```

## Summary

`innodb_file_per_table=ON` is the recommended default for most MySQL deployments. It enables disk space reclamation via `DROP TABLE`, simplifies backups, and supports per-table encryption. Use `ALTER TABLE ... ENGINE=InnoDB` to convert existing tables. For deployments with very high table counts, combine file-per-table with general tablespaces for better file management.
