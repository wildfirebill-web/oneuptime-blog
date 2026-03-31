# How to Back Up a MySQL Database with mysqldump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, mysqldump, Administration, Database, Recovery

Description: Learn how to back up a MySQL database using mysqldump, including single-database, full-server, and compressed backup strategies.

---

## Overview

`mysqldump` is the standard logical backup tool bundled with MySQL. It exports database schemas and data as a plain-text SQL file containing `CREATE TABLE` and `INSERT` statements. Logical backups are portable, easy to inspect, and can be restored to any MySQL version or compatible server.

## Prerequisites

Ensure `mysqldump` is available and you have a user with sufficient privileges:

```bash
mysqldump --version
```

The backup user needs at minimum:

```sql
GRANT SELECT, SHOW VIEW, TRIGGER, LOCK TABLES, PROCESS, RELOAD
ON *.* TO 'backup_user'@'localhost';
```

## Backing Up a Single Database

```bash
mysqldump -u backup_user -p shop > shop_backup.sql
```

You will be prompted for the password. The output file contains all DDL and data for the `shop` database.

## Backing Up Specific Tables

```bash
mysqldump -u backup_user -p shop orders customers > shop_tables.sql
```

## Backing Up All Databases

```bash
mysqldump -u backup_user -p --all-databases > all_databases.sql
```

## Including Stored Procedures, Functions, and Triggers

By default, `mysqldump` includes triggers. To also include stored procedures and functions:

```bash
mysqldump -u backup_user -p --routines --triggers shop > shop_full.sql
```

## Creating a Compressed Backup

Pipe the output directly to `gzip` to save disk space:

```bash
mysqldump -u backup_user -p --single-transaction shop | gzip > shop_backup_$(date +%F).sql.gz
```

`--single-transaction` starts a consistent read transaction for InnoDB tables without locking them, making it safe to run on a live production database.

## Common mysqldump Options

| Option | Purpose |
|---|---|
| `--single-transaction` | Consistent InnoDB backup without table locks |
| `--routines` | Include stored procedures and functions |
| `--triggers` | Include triggers (on by default) |
| `--events` | Include scheduled events |
| `--no-data` | Export schema only, no row data |
| `--no-create-info` | Export data only, no CREATE TABLE statements |
| `--where` | Filter rows with a WHERE clause |
| `--column-statistics=0` | Disable column stats (required for MySQL 8 to older server) |

## Exporting Only the Schema

```bash
mysqldump -u backup_user -p --no-data shop > shop_schema.sql
```

## Exporting Only the Data

```bash
mysqldump -u backup_user -p --no-create-info shop > shop_data.sql
```

## Automating Backups with a Shell Script

```bash
#!/usr/bin/env bash
BACKUP_DIR="/var/backups/mysql"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
USER="backup_user"
PASS="backup_password"
DB="shop"

mkdir -p "$BACKUP_DIR"

mysqldump -u "$USER" -p"$PASS" \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  "$DB" | gzip > "$BACKUP_DIR/${DB}_${DATE}.sql.gz"

# Remove backups older than 30 days
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +30 -delete

echo "Backup completed: ${DB}_${DATE}.sql.gz"
```

Add this to cron for daily execution:

```bash
0 2 * * * /usr/local/bin/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1
```

## Verifying the Backup

Check the last few lines of the SQL file for a valid completion marker:

```bash
zcat shop_backup.sql.gz | tail -5
```

A valid dump ends with lines like `-- Dump completed on 2026-03-31 02:00:01`.

## Summary

`mysqldump` creates portable logical backups of MySQL databases as SQL text files. Use `--single-transaction` for InnoDB tables to avoid locks, `--routines` to include procedures and functions, and pipe the output through `gzip` to compress large backups. Automate the process with a cron job and validate backups by checking for the completion marker in the dump file.
