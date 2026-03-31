# How to Import a TSV File into MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, TSV, Import, LOAD DATA INFILE, Data Migration

Description: Learn how to import tab-separated value (TSV) files into MySQL using LOAD DATA INFILE and mysqlimport with proper delimiter configuration.

---

## Introduction

Tab-separated value (TSV) files are common exports from spreadsheet applications, data warehouses, and reporting tools. MySQL's `LOAD DATA INFILE` handles TSV files with simple delimiter configuration, making bulk imports fast and reliable.

## Sample TSV File

A `products.tsv` file where columns are separated by tabs:

```text
id	name	category	price	stock
1	Laptop	Electronics	999.99	50
2	Mouse	Electronics	29.99	200
3	Desk	Furniture	349.00	30
```

## Creating the Target Table

```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(200),
  category VARCHAR(100),
  price DECIMAL(10,2),
  stock INT
);
```

## Importing with LOAD DATA INFILE

For a TSV file, set `FIELDS TERMINATED BY '\t'`:

```sql
LOAD DATA INFILE '/tmp/products.tsv'
INTO TABLE products
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

The `IGNORE 1 ROWS` clause skips the header line. If the file uses Windows-style line endings, use `LINES TERMINATED BY '\r\n'`.

## Importing a Local TSV File

When the file is on your local machine:

```bash
mysql --local-infile=1 -u root -p mydb
```

```sql
LOAD DATA LOCAL INFILE '/home/user/products.tsv'
INTO TABLE products
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

Enable local infile on the server if not already active:

```sql
SET GLOBAL local_infile = 1;
```

## Using mysqlimport

mysqlimport is a command-line tool that wraps `LOAD DATA INFILE`. The filename must match the table name (the extension is stripped):

```bash
mysqlimport \
  --local \
  --fields-terminated-by=$'\t' \
  --lines-terminated-by='\n' \
  --ignore-lines=1 \
  -u root -p mydb /tmp/products.tsv
```

## Handling Quoted Fields in TSV

Some TSV exporters enclose fields with double quotes for fields containing special characters:

```sql
LOAD DATA INFILE '/tmp/products_quoted.tsv'
INTO TABLE products
FIELDS TERMINATED BY '\t'
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

## Column Mapping

If the TSV column order differs from the table definition:

```sql
LOAD DATA INFILE '/tmp/products_reordered.tsv'
INTO TABLE products
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(name, category, price, stock, id);
```

## Converting TSV to CSV Before Import

If you prefer working with CSV tools:

```bash
sed 's/\t/,/g' products.tsv > products.csv
```

Then import the CSV using `LOAD DATA INFILE` with `FIELDS TERMINATED BY ','`.

## Verifying the Import

After loading, check row count and data integrity:

```sql
SELECT COUNT(*) FROM products;
SELECT * FROM products ORDER BY id LIMIT 5;
SELECT MIN(price), MAX(price), AVG(price) FROM products;
```

## Summary

TSV files import into MySQL almost identically to CSV files - the only difference is `FIELDS TERMINATED BY '\t'` instead of `FIELDS TERMINATED BY ','`. Use `LOAD DATA LOCAL INFILE` when the file is on a client machine, and always verify row counts against the source file after import to catch any truncation or encoding issues.
