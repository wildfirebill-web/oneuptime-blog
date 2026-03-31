# How to Use mysql_upgrade After a MySQL Version Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Upgrade, Database, Administration, Migration

Description: Learn how to run mysql_upgrade after upgrading MySQL to update system tables, check compatibility, and ensure your database works correctly with the new version.

---

After upgrading a MySQL server binary, you must run `mysql_upgrade` to update the system tables and data dictionary to match the new version. Skipping this step can cause errors or unexpected behavior.

## What mysql_upgrade Does

`mysql_upgrade` performs two key tasks: it checks all tables in user databases for incompatibilities with the current MySQL version, and it upgrades the system tables in the `mysql` schema. In MySQL 8.0, the upgrade process is largely automated, but understanding how it works helps with troubleshooting.

For MySQL 5.7 and earlier, the tool must be run manually after every binary upgrade. In MySQL 8.0 and later, the server performs in-place upgrade checks automatically on startup, but you may still need to handle some steps manually.

## Running mysql_upgrade on MySQL 5.7

After upgrading your MySQL 5.7 binaries and restarting the server, run the upgrade tool:

```bash
mysql_upgrade -u root -p
```

This will output a list of tables being checked and repaired. You should see output similar to:

```bash
Checking if update is needed.
Checking server version.
Running queries to upgrade MySQL server.
Checking system database.
mysql.columns_priv                                 OK
mysql.db                                           OK
mysql.user                                         OK
...
Upgrade process completed successfully.
Restarting the server, please wait...
```

## Verifying the Upgrade Was Applied

After running `mysql_upgrade`, verify that the upgrade was recorded by checking the upgrade version file:

```bash
cat /var/lib/mysql/mysql_upgrade_info
```

The file should contain the version string of the server. You can also confirm via SQL:

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'innodb_version';
```

## Upgrading with mysql_upgrade in MySQL 8.0

In MySQL 8.0, `mysql_upgrade` still exists but is effectively a no-op because the server handles upgrades during startup. However, you can still run it for compatibility:

```bash
mysql_upgrade -u root -p --upgrade-system-tables
```

The `--upgrade-system-tables` flag skips checking user tables and only upgrades the grant and system tables.

## Common Options

Use these flags to control behavior:

```bash
# Only upgrade system tables, skip user tables
mysql_upgrade --upgrade-system-tables -u root -p

# Verbose output for debugging
mysql_upgrade -u root -p --verbose

# Force upgrade even if already done
mysql_upgrade -u root -p --force

# Specify a custom socket path
mysql_upgrade -u root -p --socket=/var/run/mysqld/mysqld.sock
```

## Handling Errors During Upgrade

If `mysql_upgrade` reports errors on specific tables, you may need to repair them manually:

```sql
-- Check and repair a specific table
CHECK TABLE mydb.mytable FOR UPGRADE;
REPAIR TABLE mydb.mytable;

-- Check all tables in a database
USE mydb;
CHECK TABLE t1, t2, t3;
```

For InnoDB tables that fail compatibility checks, the usual fix is to dump and reload the table:

```bash
mysqldump -u root -p mydb mytable > mytable_backup.sql
mysql -u root -p mydb < mytable_backup.sql
```

## After the Upgrade: Flushing Privileges

After a successful upgrade, flush the privilege tables to ensure the new system table format is active:

```sql
FLUSH PRIVILEGES;
```

Then restart the MySQL service to confirm everything starts cleanly:

```bash
systemctl restart mysql
systemctl status mysql
```

## Summary

The `mysql_upgrade` tool updates MySQL system tables and checks user tables for compatibility after a binary version upgrade. In MySQL 5.7 and earlier, it must be run manually after every upgrade. In MySQL 8.0+, the server handles most of this automatically, but running the tool with `--upgrade-system-tables` is still useful for forcing privilege table updates and ensuring a clean transition.
