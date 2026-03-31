# How to Understand InnoDB Row Formats in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Storage, Row Format, Configuration

Description: Learn the differences between InnoDB row formats - REDUNDANT, COMPACT, DYNAMIC, and COMPRESSED - and how to choose the right one in MySQL.

---

## What Are InnoDB Row Formats?

InnoDB stores rows in pages using one of four row formats: REDUNDANT, COMPACT, DYNAMIC, and COMPRESSED. The row format determines how row data, variable-length columns, and overflow pages are physically stored. Choosing the right format affects storage efficiency, performance, and support for large column values.

## Checking the Default Row Format

```sql
SHOW VARIABLES LIKE 'innodb_default_row_format';
-- Default: dynamic (MySQL 5.7.9+)

-- Check a specific table's format
SHOW TABLE STATUS LIKE 'orders'\G
-- Look for Row_format field
```

## REDUNDANT (Legacy)

The original InnoDB row format, maintained for backwards compatibility:

```sql
CREATE TABLE legacy_table (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB ROW_FORMAT=REDUNDANT;
```

- Stores the lengths of all columns in each row header
- Less efficient than newer formats - not recommended for new tables
- Supported for backward compatibility only

## COMPACT

Introduced in MySQL 5.0, COMPACT reduces storage by ~20% compared to REDUNDANT:

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10,2)
) ENGINE=InnoDB ROW_FORMAT=COMPACT;
```

- Stores only a bitmask of NULL columns in the row header instead of all column lengths
- Better cache efficiency than REDUNDANT
- Overflow handling: VARCHAR/TEXT beyond 768 bytes is stored on overflow pages

## DYNAMIC (Default)

DYNAMIC is the default format since MySQL 5.7.9 and is recommended for most workloads:

```sql
CREATE TABLE documents (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    body MEDIUMTEXT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
-- This is equivalent since DYNAMIC is the default:
CREATE TABLE documents2 (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    body MEDIUMTEXT
) ENGINE=InnoDB;
```

Key differences from COMPACT:
- Variable-length columns (VARCHAR, TEXT, BLOB) can store just a 20-byte pointer in the main row, with the full value on an overflow page
- No 768-byte inline prefix requirement
- Much better performance for large TEXT/BLOB columns

## COMPRESSED

COMPRESSED uses zlib compression on each page:

```sql
CREATE TABLE archive_orders (
    id INT PRIMARY KEY,
    order_data JSON,
    created_at DATETIME
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED
  KEY_BLOCK_SIZE=8;  -- Compressed page size in KB (4 or 8, must be <= innodb_page_size/2)
```

- Reduces storage by 50-80% for compressible data
- Requires more CPU for compression/decompression
- Higher memory usage (both compressed and uncompressed pages may be in buffer pool)
- Not supported in InnoDB temporary tablespace

## Comparing Row Formats

```sql
-- See format and row count for all tables
SELECT
    TABLE_NAME,
    ROW_FORMAT,
    TABLE_ROWS,
    DATA_LENGTH / 1024 / 1024 AS data_mb,
    AVG_ROW_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND ENGINE = 'InnoDB'
ORDER BY DATA_LENGTH DESC;
```

## Changing Row Format on Existing Tables

```sql
-- Change row format (causes full table rebuild)
ALTER TABLE orders ROW_FORMAT=DYNAMIC;

-- Check progress for large tables
SHOW PROCESSLIST;
```

An ALTER TABLE to change row format rebuilds the entire table and temporarily requires 2x disk space.

## Recommendations

```text
Format      | Use Case
------------|--------------------------------------------------
DYNAMIC     | Default, best for most tables with TEXT/BLOB
COMPACT     | Legacy compatibility only
COMPRESSED  | Archive tables, large JSON/TEXT, I/O-bound systems
REDUNDANT   | Never use for new tables
```

## Summary

InnoDB supports four row formats: REDUNDANT (legacy), COMPACT (legacy improvement), DYNAMIC (default since MySQL 5.7.9), and COMPRESSED (zlib-compressed pages). DYNAMIC is recommended for most workloads as it efficiently handles large variable-length columns by storing overflow data on separate pages. COMPRESSED is valuable for archive tables or I/O-constrained environments where CPU for decompression is available.
