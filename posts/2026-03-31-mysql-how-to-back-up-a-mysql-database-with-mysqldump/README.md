# How to Back Up a MySQL Database with mysqldump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Backup, Mysqldump, Administration, Disaster Recovery

Description: Learn how to use mysqldump to create logical backups of MySQL databases, including full database, single table, and compressed backup options.

---

## What Is mysqldump?

`mysqldump` is a command-line utility included with MySQL that creates logical backups. It generates a SQL script containing `CREATE TABLE` and `INSERT` statements (or `LOAD DATA` infile references) that can be used to recreate the database.

Logical backups are portable, human-readable, and can be used to migrate between MySQL versions. For large databases, consider physical backup tools like Percona XtraBackup for faster backup and restore.

## Backing Up a Single Database

```bash
mysqldump -u root -p your_database > backup.sql
```

This prompts for the root password and writes the SQL dump to `backup.sql`.

## Backing Up with Connection Options

```bash
mysqldump \
  --host=127.0.0.1 \
  --port=3306 \
  --user=backup_user \
  --password=BackupPass! \
  your_database > backup.sql
```

## Backing Up All Databases

```bash
mysqldump --all-databases -u root -p > all_databases.sql
```

## Backing Up Multiple Specific Databases

```bash
mysqldump --databases db1 db2 db3 -u root -p > multi_db_backup.sql
```

## Backing Up a Single Table

```bash
mysqldump -u root -p your_database orders > orders_backup.sql
```

## Compressed Backup

Pipe through `gzip` to save space:

```bash
mysqldump -u root -p your_database | gzip > backup_$(date +%Y%m%d).sql.gz
```

## Recommended Flags for InnoDB Tables

```bash
mysqldump \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  -u root -p \
  your_database > backup.sql
```

| Flag | Purpose |
|---|---|
| `--single-transaction` | Takes a consistent snapshot without locking tables (InnoDB only) |
| `--routines` | Includes stored procedures and functions |
| `--triggers` | Includes triggers (included by default, explicit is safer) |
| `--events` | Includes scheduled events |

## Locking for MyISAM Tables

MyISAM tables do not support transactions. Use `--lock-tables` instead:

```bash
mysqldump --lock-tables -u root -p myisam_database > backup.sql
```

`--lock-all-tables` (`-x`) locks all tables across all databases for a fully consistent dump of mixed-engine environments.

## Including CREATE DATABASE Statement

By default, `mysqldump` does not include `CREATE DATABASE` or `USE` statements unless `--databases` or `--all-databases` is used:

```bash
# Includes CREATE DATABASE
mysqldump --databases your_database -u root -p > backup.sql

# Does NOT include CREATE DATABASE (restore into any existing database)
mysqldump your_database -u root -p > backup.sql
```

## Schema-Only Backup (No Data)

```bash
mysqldump --no-data -u root -p your_database > schema_only.sql
```

## Data-Only Backup (No Schema)

```bash
mysqldump --no-create-info -u root -p your_database > data_only.sql
```

## Automating Backups with Cron

```bash
crontab -e
```

Add a daily backup at 2 AM:

```text
0 2 * * * mysqldump --single-transaction --routines --triggers --events -u backup_user -pBackupPass! your_database | gzip > /backups/db_$(date +\%Y\%m\%d).sql.gz
```

Create a dedicated backup user with minimal privileges:

```sql
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'BackupPass!';
GRANT SELECT, SHOW VIEW, TRIGGER, EVENT, LOCK TABLES, RELOAD
  ON *.* TO 'backup_user'@'localhost';
```

## Summary

`mysqldump` creates portable SQL-based logical backups of MySQL databases. Use `--single-transaction` for consistent InnoDB backups without locking, and combine with `--routines`, `--triggers`, and `--events` to capture all database objects. Pipe through `gzip` for compressed backups, and use a dedicated backup user with minimal privileges. For large databases requiring fast backup and restore, consider Percona XtraBackup.
