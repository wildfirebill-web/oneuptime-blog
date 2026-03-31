# How to Use REPAIR TABLE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database Maintenance, Table Repair, MyISAM, Storage Engine

Description: Learn how to use REPAIR TABLE in MySQL to fix corrupted MyISAM and ARCHIVE tables, with syntax, options, and practical recovery examples.

---

## What Is REPAIR TABLE?

The `REPAIR TABLE` statement in MySQL repairs corrupted table files. It is primarily used with **MyISAM** and **ARCHIVE** storage engines. InnoDB tables do not support `REPAIR TABLE` directly - for InnoDB, you should use crash recovery or `ALTER TABLE ... ENGINE=InnoDB` instead.

A table may become corrupted due to:
- Unexpected server shutdown
- Disk I/O errors
- Hardware failures
- File system corruption

## Basic Syntax

```sql
REPAIR TABLE table_name [QUICK] [EXTENDED] [USE_FRM];
```

You can repair multiple tables in one statement:

```sql
REPAIR TABLE table1, table2, table3;
```

## REPAIR TABLE Options

| Option | Description |
|--------|-------------|
| `QUICK` | Only repairs the index file, not the data file. Faster but less thorough. |
| `EXTENDED` | Creates the index row by row instead of with one sort pass. Slower but handles more cases. |
| `USE_FRM` | Uses the `.frm` file to recreate the index file when the index is missing or corrupted. |

## Checking If a Table Needs Repair

Before running a repair, use `CHECK TABLE` to identify corruption:

```sql
CHECK TABLE orders;
```

If the output shows `status: error` or `Corrupt`, then a repair is needed.

## Running a Basic Repair

```sql
REPAIR TABLE orders;
```

Example output:

```text
+------------------+--------+----------+----------+
| Table            | Op     | Msg_type | Msg_text |
+------------------+--------+----------+----------+
| mydb.orders      | repair | status   | OK       |
+------------------+--------+----------+----------+
```

If the repair succeeded, you will see `status: OK`.

## Quick Repair

Use `QUICK` when only the index is damaged and the data rows are intact:

```sql
REPAIR TABLE orders QUICK;
```

This is significantly faster for large tables because it skips rebuilding the data file.

## Extended Repair

Use `EXTENDED` when `QUICK` is not sufficient:

```sql
REPAIR TABLE orders EXTENDED;
```

This rebuilds indexes row by row and is more thorough. It handles more complex corruption scenarios but takes longer.

## Repair Using FRM File

If the index file is missing or severely corrupted, use `USE_FRM`:

```sql
REPAIR TABLE orders USE_FRM;
```

This reconstructs the index using the `.frm` file definition. Use this as a last resort.

## Repairing Tables from the Command Line

You can also repair MyISAM tables using the `myisamchk` utility when the MySQL server is stopped:

```bash
# Stop MySQL first
sudo systemctl stop mysql

# Repair the table
myisamchk --recover /var/lib/mysql/mydb/orders.MYI

# Start MySQL again
sudo systemctl start mysql
```

Or use `mysqlcheck` while the server is running:

```bash
mysqlcheck --repair --databases mydb
```

Repair all databases:

```bash
mysqlcheck --repair --all-databases -u root -p
```

## Checking Repair Results Programmatically

You can store the result of `REPAIR TABLE` in a temporary table to check it programmatically:

```sql
CREATE TEMPORARY TABLE repair_result
SELECT * FROM (REPAIR TABLE orders) AS r;

SELECT Msg_text FROM repair_result WHERE Msg_type = 'error';
```

## Handling InnoDB Tables

`REPAIR TABLE` does not work on InnoDB tables. For InnoDB corruption, options include:

```sql
-- Force InnoDB recovery mode via my.cnf
-- [mysqld]
-- innodb_force_recovery = 1

-- Then dump and reimport data
-- mysqldump -u root -p mydb orders > orders_backup.sql
-- mysql -u root -p mydb < orders_backup.sql
```

An alternative is to rebuild the table:

```sql
ALTER TABLE orders ENGINE = InnoDB;
```

## Automating Repair in a Script

```bash
#!/bin/bash
DB="mydb"
TABLES=$(mysql -u root -p"$MYSQL_PASS" -N -e "SHOW TABLES FROM $DB;")

for TABLE in $TABLES; do
  RESULT=$(mysql -u root -p"$MYSQL_PASS" -N -e "REPAIR TABLE $DB.$TABLE;" | awk '{print $NF}')
  echo "$TABLE: $RESULT"
done
```

## Summary

`REPAIR TABLE` is a built-in MySQL command for recovering corrupted MyISAM and ARCHIVE tables. It offers three modes - `QUICK` for fast index repair, `EXTENDED` for thorough row-by-row index rebuilding, and `USE_FRM` for reconstructing from the schema definition. For InnoDB tables, use crash recovery or `ALTER TABLE` to rebuild instead.
