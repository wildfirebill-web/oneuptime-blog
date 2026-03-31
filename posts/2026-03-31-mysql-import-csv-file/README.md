# How to Import a CSV File into MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CSV, Import, LOAD DATA INFILE, Data Migration

Description: Learn how to import a CSV file into a MySQL table using LOAD DATA INFILE and the MySQL command-line client for fast, reliable bulk data loading.

---

## Introduction

Importing CSV data into MySQL is a common task for data migrations, ETL pipelines, and seeding databases. MySQL provides `LOAD DATA INFILE` as the fastest way to bulk-load CSV files directly into a table, bypassing row-by-row INSERT overhead.

## Preparing the CSV File

Ensure your CSV has consistent formatting. A sample `customers.csv`:

```text
id,name,email,created_at
1,Alice Smith,alice@example.com,2025-01-01
2,Bob Jones,bob@example.com,2025-01-02
3,Carol White,carol@example.com,2025-01-03
```

## Creating the Target Table

```sql
CREATE TABLE customers (
  id INT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(150) NOT NULL,
  created_at DATE
);
```

## Importing with LOAD DATA INFILE

```sql
LOAD DATA INFILE '/tmp/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

- `FIELDS TERMINATED BY ','` - comma delimiter
- `OPTIONALLY ENCLOSED BY '"'` - handles quoted fields
- `IGNORE 1 ROWS` - skips the header line

## Using LOCAL for Client-Side Files

If the CSV is on your local machine rather than the server:

```bash
mysql --local-infile=1 -u root -p mydb
```

Then in the MySQL session:

```sql
LOAD DATA LOCAL INFILE '/home/user/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

The server must also have `local_infile` enabled:

```sql
SET GLOBAL local_infile = 1;
```

## Importing via mysqlimport

mysqlimport is a command-line wrapper around `LOAD DATA INFILE`. The filename (minus extension) must match the table name:

```bash
mysqlimport \
  --local \
  --fields-terminated-by=',' \
  --fields-optionally-enclosed-by='"' \
  --lines-terminated-by='\n' \
  --ignore-lines=1 \
  -u root -p mydb /tmp/customers.csv
```

## Mapping Columns from CSV

If your CSV column order does not match the table, specify the mapping explicitly:

```sql
LOAD DATA INFILE '/tmp/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(id, name, email, created_at);
```

## Verifying the Import

After loading, verify row counts and spot-check data:

```sql
SELECT COUNT(*) FROM customers;
SELECT * FROM customers LIMIT 5;
```

## Summary

`LOAD DATA INFILE` is the most efficient method for importing CSV files into MySQL, orders of magnitude faster than individual INSERT statements. Use `LOAD DATA LOCAL INFILE` when the file lives on the client machine, and `mysqlimport` for a convenient command-line interface. Always test on a small subset before loading production data.
