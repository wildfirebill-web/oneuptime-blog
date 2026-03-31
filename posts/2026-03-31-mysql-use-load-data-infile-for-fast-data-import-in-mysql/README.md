# How to Use LOAD DATA INFILE for Fast Data Import in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Load Data Infile, Bulk Import, Csv, Database

Description: Learn how to use LOAD DATA INFILE in MySQL to import large CSV or delimited files into tables at high speed, with options for custom formats and error handling.

---

## Introduction

`LOAD DATA INFILE` is MySQL's fastest method for bulk importing data from a flat file into a table. It is significantly faster than individual `INSERT` statements because it reads the file directly, bypasses some row-by-row overhead, and uses optimized internal loading mechanisms.

## Basic Syntax

```sql
LOAD DATA INFILE '/path/to/file.csv'
INTO TABLE table_name
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(column1, column2, column3);
```

## Prerequisites

- The file must be accessible by the MySQL server process.
- The user needs the `FILE` privilege.
- `secure_file_priv` must allow the directory (or be empty to allow any path).

Check the allowed path:

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

## Basic CSV Import

Given a file `/var/lib/mysql-files/employees.csv`:

```text
1,Alice Johnson,alice@example.com,Engineering,75000
2,Bob Smith,bob@example.com,Marketing,55000
```

Import it:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/employees.csv'
INTO TABLE employees
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, name, email, department, salary);
```

## Skipping Header Rows

If the CSV has a header row, use `IGNORE`:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/employees.csv'
INTO TABLE employees
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(id, name, email, department, salary);
```

## Handling Quoted Fields

For CSV files with quoted strings:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(id, first_name, last_name, email, city);
```

## Custom Line Terminator

For Windows-formatted files (CRLF line endings):

```sql
LOAD DATA INFILE '/var/lib/mysql-files/data.csv'
INTO TABLE records
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n'
(id, value);
```

## Tab-Separated Files

```sql
LOAD DATA INFILE '/var/lib/mysql-files/data.tsv'
INTO TABLE records
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
(id, name, value);
```

## Column Transformations with SET

Use `SET` to transform data during import:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(id, @raw_name, email, @raw_date)
SET
  name = TRIM(@raw_name),
  created_at = STR_TO_DATE(@raw_date, '%m/%d/%Y');
```

Here `@raw_name` and `@raw_date` are user variables used as intermediate values.

## Handling NULL Values

By default, `\N` in the file is treated as NULL. Customize with `NULL DEFINED BY`:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/data.csv'
INTO TABLE orders
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, customer_id, @notes)
SET notes = NULLIF(@notes, '');
```

## Replace vs Ignore on Duplicate Keys

```sql
-- Replace existing rows on duplicate key
LOAD DATA INFILE '/data/products.csv'
REPLACE INTO TABLE products
FIELDS TERMINATED BY ','
(sku, name, price);

-- Skip rows that conflict with existing unique keys
LOAD DATA INFILE '/data/products.csv'
IGNORE INTO TABLE products
FIELDS TERMINATED BY ','
(sku, name, price);
```

## LOAD DATA LOCAL INFILE

To load a file from the client machine (not the server):

```sql
LOAD DATA LOCAL INFILE '/home/user/data.csv'
INTO TABLE table_name
FIELDS TERMINATED BY ',';
```

This requires `local_infile = ON` on both server and client.

## Performance Optimization

For maximum speed during bulk loading:

```sql
-- Disable checks temporarily
SET foreign_key_checks = 0;
SET unique_checks = 0;

LOAD DATA INFILE '/var/lib/mysql-files/large_data.csv'
INTO TABLE large_table
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';

-- Re-enable
SET unique_checks = 1;
SET foreign_key_checks = 1;

-- Rebuild indexes
ANALYZE TABLE large_table;
```

## Summary

`LOAD DATA INFILE` is the fastest native way to bulk import data into MySQL from files. It supports CSV, TSV, and custom delimited formats, handles headers, allows column mapping, and can transform data during load. For large datasets, disable foreign key and unique checks temporarily to maximize throughput.
