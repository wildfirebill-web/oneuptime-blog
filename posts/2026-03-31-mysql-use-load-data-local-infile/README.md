# How to Use LOAD DATA LOCAL INFILE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, LOAD DATA, Import, CSV, Bulk Insert

Description: Learn how to use LOAD DATA LOCAL INFILE to import CSV and delimited files from the client machine into a MySQL table quickly.

---

## What Is LOAD DATA LOCAL INFILE

`LOAD DATA LOCAL INFILE` imports a text file from the client machine (where the MySQL client is running) into a MySQL table. Unlike `LOAD DATA INFILE`, which reads from the server's filesystem, the `LOCAL` keyword instructs the client to send the file over the connection. This makes it usable without server-side file access or the `FILE` privilege.

## Enabling LOCAL INFILE

The `local_infile` option must be enabled on both the server and the client.

On the server (in `my.cnf`):

```bash
[mysqld]
local_infile = 1
```

In a client session:

```sql
SET GLOBAL local_infile = 1;
```

When connecting from the command line, pass the `--local-infile` flag:

```bash
mysql --local-infile=1 -u root -p mydb
```

## Basic Import from a CSV File

Suppose you have a CSV file at `/home/user/customers.csv`:

```
1,Alice,alice@example.com,New York
2,Bob,bob@example.com,Chicago
3,Carol,carol@example.com,Los Angeles
```

Import it into the `customers` table:

```sql
LOAD DATA LOCAL INFILE '/home/user/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, name, email, city);
```

## Handling Headers

If the CSV has a header row, skip it with `IGNORE 1 ROWS`:

```sql
LOAD DATA LOCAL INFILE '/home/user/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(id, name, email, city);
```

## Setting Default Values for Missing Columns

Use a variable placeholder to discard unwanted columns or set computed values:

```sql
LOAD DATA LOCAL INFILE '/home/user/orders.csv'
INTO TABLE orders
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(id, user_id, amount, @created_str)
SET created_at = STR_TO_DATE(@created_str, '%Y-%m-%d');
```

## Importing from an Application

Most MySQL drivers support `LOAD DATA LOCAL INFILE` via the client API:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='app',
    password='secret',
    database='mydb',
    allow_local_infile=True
)

cursor = conn.cursor()
cursor.execute("""
    LOAD DATA LOCAL INFILE '/tmp/data.csv'
    INTO TABLE records
    FIELDS TERMINATED BY ','
    LINES TERMINATED BY '\\n'
    IGNORE 1 ROWS
""")
conn.commit()
```

## Security Considerations

`LOCAL INFILE` transfers a file from the client to the server. Using it against an untrusted server carries a risk: the server can request any file accessible to the client. Only use `LOCAL INFILE` against servers you control or trust.

## Performance Tips

- Wrap the import in a transaction and disable indexes temporarily for very large files.
- Use `ALTER TABLE ... DISABLE KEYS` before loading and `ENABLE KEYS` after.
- Consider `SET FOREIGN_KEY_CHECKS = 0` during bulk imports if referential integrity is validated separately.

## Summary

`LOAD DATA LOCAL INFILE` is a fast and flexible way to import delimited files from the client machine into MySQL without requiring server-side file access. Enable `local_infile` on both server and client, handle headers with `IGNORE N ROWS`, and use variable assignments for data transformations during import.
