# How to Fix ERROR 1016 Can't Open File in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, File, InnoDB, Permission

Description: Fix MySQL ERROR 1016 Can't open file by repairing table files, fixing permissions, recovering InnoDB tablespaces, and resolving open_files_limit constraints.

---

MySQL ERROR 1016 occurs when MySQL cannot open a table's data file. The message reads: `ERROR 1016 (HY000): Can't open file: 'tablename.MYI' (errno: 145)`. The errno value identifies the specific cause. This error is more common with MyISAM tables but can also occur with InnoDB.

## Identify the Error Number

The errno value indicates the problem:
- `errno: 2` - File not found (deleted or moved)
- `errno: 13` - Permission denied
- `errno: 23` or `errno: 24` - Too many open files
- `errno: 145` - Table was not properly closed (MyISAM crash)

```bash
# Check the MySQL error log for more details
sudo tail -100 /var/log/mysql/error.log
```

## Fix: Repair a Crashed MyISAM Table

For MyISAM tables with `errno: 145`:

```sql
-- Use CHECK TABLE to diagnose
CHECK TABLE your_table_name;

-- Repair the table
REPAIR TABLE your_table_name;

-- Or use QUICK mode first
REPAIR TABLE your_table_name QUICK;
```

From the command line when MySQL is stopped:

```bash
myisamchk -r /var/lib/mysql/dbname/tablename.MYI
```

## Fix: File Not Found

If the `.frm`, `.MYD`, or `.MYI` files are missing, you need to restore from backup:

```bash
# Check if files exist
ls -la /var/lib/mysql/dbname/

# Look for recent backups
ls -lt /var/backups/mysql/
```

If the `.frm` file exists but `.MYD` is missing, recreate the data file:

```sql
-- This is destructive - data will be lost
CREATE TABLE new_tablename LIKE your_table_name;
RENAME TABLE your_table_name TO old_backup, new_tablename TO your_table_name;
DROP TABLE old_backup;
```

## Fix: Permission Issues

MySQL must be able to read and write the table files:

```bash
# Check file ownership
ls -la /var/lib/mysql/dbname/

# Fix permissions - mysql user must own all data files
sudo chown -R mysql:mysql /var/lib/mysql/
sudo chmod 660 /var/lib/mysql/dbname/*.MYD
sudo chmod 660 /var/lib/mysql/dbname/*.MYI
sudo chmod 660 /var/lib/mysql/dbname/*.frm
```

## Fix: Too Many Open Files

If `errno: 23` or `errno: 24` appears, MySQL has hit the OS file descriptor limit:

```sql
SHOW VARIABLES LIKE 'open_files_limit';
SHOW GLOBAL STATUS LIKE 'Open_files';
```

Increase the limit in `my.cnf`:

```text
[mysqld]
open_files_limit = 65535
```

Also increase the OS limit:

```bash
# Check current limit
ulimit -n

# Increase for the mysql service in systemd
sudo systemctl edit mysql
```

```text
[Service]
LimitNOFILE=65535
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart mysql
```

## Fix: InnoDB Missing .ibd File

For InnoDB with `innodb_file_per_table`, the `.ibd` file contains the table data:

```sql
-- Discard and reimport the tablespace
ALTER TABLE your_table_name DISCARD TABLESPACE;
-- Copy the .ibd file from backup to the database directory
ALTER TABLE your_table_name IMPORT TABLESPACE;
```

## Summary

ERROR 1016 diagnosis starts with the errno code. For MyISAM crashes use `REPAIR TABLE`, for permission issues fix file ownership, and for too-many-open-files increase `open_files_limit`. InnoDB tablespace issues require tablespace discard and import from backup. Always check the MySQL error log for full context around this error.
