# How to Troubleshoot MySQL Import Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Import, mysqldump, LOAD DATA, Troubleshooting

Description: Learn how to diagnose and fix MySQL import errors including foreign key violations, character encoding issues, packet size limits, and lock timeouts.

---

## Common Import Failure Types

Imports fail for several reasons: constraint violations, encoding mismatches, oversized packets, or running out of disk space mid-import. The error message usually points directly to the cause.

## ERROR 1005: Can't Create Table (Foreign Key Constraint)

```text
ERROR 1005 (HY000): Can't create table `myapp`.`orders` (errno: 150 "Foreign key constraint is incorrectly formed")
```

This happens when the referenced table does not exist yet during import. Disable foreign key checks for the duration of the import:

```sql
SET foreign_key_checks = 0;
SOURCE /tmp/backup.sql;
SET foreign_key_checks = 1;
```

Or on the command line:

```bash
mysql -u root -p myapp --init-command="SET foreign_key_checks=0;" < /tmp/backup.sql
```

## ERROR 1153: Got a Packet Bigger Than max_allowed_packet

```text
ERROR 1153 (08S01): Got a packet bigger than 'max_allowed_packet' bytes
```

Increase `max_allowed_packet` for the import session or globally:

```sql
SET GLOBAL max_allowed_packet = 1073741824; -- 1 GB
```

Or pass it as a client option:

```bash
mysql -u root -p --max_allowed_packet=1G myapp < /tmp/backup.sql
```

## ERROR 1366: Incorrect String Value (Encoding Mismatch)

```text
ERROR 1366 (HY000): Incorrect string value: '\xE2\x80\x9C' for column 'name'
```

The dump was made from a `utf8mb4` database but the target table uses `latin1`. Check the target database character set:

```sql
SELECT DEFAULT_CHARACTER_SET_NAME FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'myapp';
```

Ensure the target database uses `utf8mb4` before importing:

```sql
ALTER DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Add the `--default-character-set=utf8mb4` flag to the import:

```bash
mysql -u root -p --default-character-set=utf8mb4 myapp < /tmp/backup.sql
```

## Stalled or Slow Imports

Large imports can appear stalled. Monitor progress:

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
```

Speed up imports by temporarily disabling safety checks:

```sql
SET unique_checks = 0;
SET foreign_key_checks = 0;
SET sql_log_bin = 0;  -- Only on standalone servers, not replicas
```

Alternatively, use `mysqlimport` or `LOAD DATA INFILE` for CSV files, which is significantly faster than INSERT-based dumps.

## LOAD DATA INFILE Permission Errors

```text
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option
```

Check where MySQL allows file reads:

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

Move the file to that directory, or set `secure_file_priv = ""` in `my.cnf` (use with caution):

```bash
mysql -u root -p myapp \
  -e "LOAD DATA INFILE '/var/lib/mysql-files/data.csv'
      INTO TABLE orders
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      IGNORE 1 ROWS;"
```

## Disk Full During Import

```text
ERROR 3 (HY000): Error writing file '/var/lib/mysql/myapp/orders.ibd' (Errcode: 28 - No space left on device)
```

Check disk usage and free space before starting large imports:

```bash
df -h /var/lib/mysql
```

Consider importing in chunks using multiple smaller SQL files split from the original dump.

## Summary

MySQL import errors are usually caused by foreign key ordering (disable with `SET foreign_key_checks=0`), undersized `max_allowed_packet`, encoding mismatches between source and target, or insufficient disk space. Always check encoding settings match between the source dump and target database. For large imports, disable unique checks and foreign key checks temporarily and monitor disk usage throughout the process.
