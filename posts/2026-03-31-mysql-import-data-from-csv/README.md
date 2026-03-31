# How to Import Data from CSV in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CSV, Import, LOAD DATA INFILE, Data

Description: Learn how to import CSV data into MySQL tables using LOAD DATA INFILE and mysqlimport for fast bulk data loading from flat files.

---

Importing CSV data into MySQL is a common task for bulk data loading, data migration, and ETL pipelines. MySQL's `LOAD DATA INFILE` statement is the fastest way to import CSV files - it is orders of magnitude faster than inserting rows individually because it bypasses row-by-row SQL processing.

## Method 1: LOAD DATA INFILE (Fastest)

```sql
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

`IGNORE 1 ROWS` skips the header row if present.

## Specifying Column Order

If the CSV columns are in a different order than the table columns:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(id, full_name, email, created_at);
```

## Transforming Data During Import

Use `SET` to transform values on load:

```sql
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(full_name, email, @raw_date)
SET
  created_at = STR_TO_DATE(@raw_date, '%m/%d/%Y'),
  active = 1;
```

## Configuring secure_file_priv

The file must be in the allowed directory:

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

Copy your CSV to the allowed path:

```bash
cp /local/users.csv /var/lib/mysql-files/users.csv
chown mysql:mysql /var/lib/mysql-files/users.csv
```

## Method 2: LOAD DATA LOCAL INFILE (Client-Side File)

`LOCAL` reads the file from the client machine instead of the server:

```sql
LOAD DATA LOCAL INFILE '/home/user/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

Enable `LOCAL` in the client connection:

```bash
mysql -u root -p --local-infile=1 myapp
```

And in `my.cnf`:

```text
[mysqld]
local_infile=ON
```

## Method 3: mysqlimport (Command-Line Tool)

`mysqlimport` is a command-line wrapper around `LOAD DATA INFILE`. The file name (without extension) must match the table name:

```bash
mysqlimport -u root -p \
  --fields-terminated-by=',' \
  --fields-optionally-enclosed-by='"' \
  --lines-terminated-by='\n' \
  --ignore-lines=1 \
  myapp \
  /var/lib/mysql-files/users.csv
```

## Handling Duplicate Keys

```sql
-- Replace existing rows with same primary key
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
REPLACE INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Ignore rows that violate unique constraints
LOAD DATA INFILE '/var/lib/mysql-files/users.csv'
IGNORE INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

## Performance Tips for Large Imports

```sql
-- Disable indexes before bulk import, rebuild after
ALTER TABLE users DISABLE KEYS;

LOAD DATA INFILE '/var/lib/mysql-files/large_users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

ALTER TABLE users ENABLE KEYS;
```

Also increase `innodb_buffer_pool_size` temporarily and set `unique_checks=0` and `foreign_key_checks=0` for maximum speed.

## Verifying the Import

```sql
-- Check row count after import
SELECT COUNT(*) FROM users;

-- Spot-check data
SELECT * FROM users ORDER BY id DESC LIMIT 5;
```

## Summary

`LOAD DATA INFILE` is the most efficient way to import CSV data into MySQL, using server-side file access for maximum throughput. Use `LOCAL INFILE` for client-side files, specify column mappings and `SET` expressions for data transformation, and disable/enable indexes around large bulk loads. Always verify row counts and spot-check data after import to confirm the file was read correctly.
