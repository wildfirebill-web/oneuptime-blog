# How to Use CREATE TABLESPACE Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Tablespace, InnoDB, Storage, DDL

Description: Learn how to use CREATE TABLESPACE in MySQL to define custom InnoDB general tablespaces for flexible, shared storage management.

---

## Overview

The `CREATE TABLESPACE` statement in MySQL creates a new InnoDB general tablespace - a shared storage container that can hold data from multiple tables. General tablespaces give you control over where table data is physically stored, allowing you to distribute tables across different disk volumes or storage tiers.

## Basic Syntax

```sql
CREATE TABLESPACE tablespace_name
  ADD DATAFILE 'file_path'
  [FILE_BLOCK_SIZE = block_size]
  [ENGINE = InnoDB];
```

The `ADD DATAFILE` clause specifies the path to the `.ibd` data file. The path can be absolute or relative to the MySQL data directory.

## Creating a General Tablespace

```sql
-- Create a tablespace in the default data directory
CREATE TABLESPACE ts_general
  ADD DATAFILE 'ts_general.ibd'
  ENGINE = InnoDB;
```

```sql
-- Create a tablespace on a separate disk
CREATE TABLESPACE ts_fast_disk
  ADD DATAFILE '/mnt/ssd/mysql/ts_fast.ibd'
  ENGINE = InnoDB;
```

## Assigning Tables to a Tablespace

After creating a tablespace, you can assign tables to it with `CREATE TABLE ... TABLESPACE`:

```sql
CREATE TABLE orders (
  id         INT          NOT NULL AUTO_INCREMENT,
  customer   VARCHAR(100) NOT NULL,
  total      DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (id)
) TABLESPACE ts_fast_disk;
```

You can also move an existing table to a different tablespace with `ALTER TABLE`:

```sql
ALTER TABLE orders TABLESPACE ts_general;
```

## Setting the File Block Size

The `FILE_BLOCK_SIZE` option sets the page size for the tablespace. It must match the `innodb_page_size` system variable for row-format tables:

```sql
CREATE TABLESPACE ts_compressed
  ADD DATAFILE 'ts_compressed.ibd'
  FILE_BLOCK_SIZE = 8192
  ENGINE = InnoDB;
```

File block sizes of 4096 or 8192 enable compressed row formats in the tablespace.

## Encrypted Tablespace

On MySQL 8.0, you can create an encrypted tablespace if the keyring plugin is installed:

```sql
CREATE TABLESPACE ts_secure
  ADD DATAFILE 'ts_secure.ibd'
  ENCRYPTION = 'Y'
  ENGINE = InnoDB;
```

## Viewing Existing Tablespaces

```sql
SELECT TABLESPACE_NAME, FILE_NAME, TOTAL_EXTENTS, EXTENT_SIZE
FROM INFORMATION_SCHEMA.FILES
WHERE FILE_TYPE = 'TABLESPACE';
```

Or check `INFORMATION_SCHEMA.INNODB_TABLESPACES`:

```sql
SELECT NAME, ROW_FORMAT, PAGE_SIZE, ENCRYPTION
FROM INFORMATION_SCHEMA.INNODB_TABLESPACES
WHERE NAME LIKE 'ts_%';
```

## Dropping a Tablespace

Before dropping a tablespace, all tables using it must be moved to a different tablespace or dropped:

```sql
ALTER TABLE orders TABLESPACE innodb_file_per_table;
DROP TABLESPACE ts_fast_disk;
```

## Summary

`CREATE TABLESPACE` in MySQL defines a named InnoDB general tablespace backed by a physical data file. It enables you to place multiple tables in a single shared file, move tables between storage volumes, and optionally enable encryption at the tablespace level. General tablespaces complement the default `innodb_file_per_table` configuration by offering a middle ground between the system tablespace and per-table files.
