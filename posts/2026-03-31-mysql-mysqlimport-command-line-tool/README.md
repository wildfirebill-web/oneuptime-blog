# How to Use mysqlimport Command-Line Tool

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Import, CSV, Command Line

Description: Learn how to use the mysqlimport command-line tool to bulk-load CSV and delimited text files into MySQL tables quickly and efficiently.

---

## What Is mysqlimport?

`mysqlimport` is a MySQL command-line utility that provides a command-line interface to the `LOAD DATA INFILE` SQL statement. It loads data from text files (CSV, TSV, or any delimiter-separated format) directly into MySQL tables, and is significantly faster than row-by-row `INSERT` statements for bulk data loading.

The tool determines the target table from the filename: a file named `orders.csv` is loaded into the `orders` table.

## Basic Syntax

```bash
mysqlimport [options] database_name file1 [file2 ...]
```

## Simple Example

Load a CSV file into the `customers` table in the `mydb` database:

```bash
# customers.csv must match the customers table name
mysqlimport -u root -p mydb /tmp/customers.csv
```

The file `/tmp/customers.csv` is loaded into the `customers` table.

## Specifying the Delimiter

```bash
# Load a tab-delimited file
mysqlimport --fields-terminated-by='\t' -u root -p mydb /tmp/orders.tsv

# Load a pipe-delimited file
mysqlimport --fields-terminated-by='|' -u root -p mydb /tmp/products.txt

# Load with quoted fields
mysqlimport \
  --fields-terminated-by=',' \
  --fields-enclosed-by='"' \
  -u root -p mydb /tmp/customers.csv
```

## Handling Lines and Rows

```bash
# Skip the header row (1 line)
mysqlimport --ignore-lines=1 \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv

# Custom line terminator (e.g., Windows CRLF)
mysqlimport --lines-terminated-by='\r\n' \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv
```

## Importing into Specific Columns

```bash
# Specify column order when file columns differ from table columns
mysqlimport \
  --columns=id,name,email,created_at \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv
```

## Using --replace and --ignore

```bash
# Replace existing rows with matching primary key
mysqlimport --replace \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv

# Skip rows that would cause duplicate key errors
mysqlimport --ignore \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv
```

## Parallel Loading

```bash
# Load multiple files in parallel (each in its own thread)
mysqlimport --use-threads=4 \
  --fields-terminated-by=',' \
  -u root -p mydb \
  /tmp/orders_jan.csv \
  /tmp/orders_feb.csv \
  /tmp/orders_mar.csv \
  /tmp/orders_apr.csv
```

## Equivalent LOAD DATA INFILE

`mysqlimport` generates `LOAD DATA INFILE` statements internally. The equivalent SQL for a basic CSV import is:

```sql
LOAD DATA INFILE '/tmp/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(id, name, email, created_at);
```

## Troubleshooting Permissions

```bash
# If you get an access denied error, use LOCAL
mysqlimport --local \
  --fields-terminated-by=',' \
  -u root -p mydb /tmp/customers.csv
```

Note: `--local` reads the file on the client rather than the server. Enable it in the server config:

```text
[mysqld]
local_infile=1
```

## Summary

`mysqlimport` is the fastest and simplest way to bulk-load text files into MySQL tables from the command line. Use `--fields-terminated-by` and `--fields-enclosed-by` to match your file format, `--ignore-lines` to skip headers, and `--use-threads` for parallel loads. For maximum performance, disable indexes before loading and rebuild them afterward using `ALTER TABLE ... DISABLE KEYS`.
