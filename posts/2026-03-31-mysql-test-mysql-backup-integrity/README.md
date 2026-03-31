# How to Test MySQL Backup Integrity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, Testing, Recovery, Database

Description: Learn how to verify MySQL backup integrity by performing test restores, checksums, and validation queries on backup files.

---

A backup you have never tested is not a backup - it is a false sense of security. Many teams discover their backups are corrupt or incomplete only when they need them most. Regularly testing MySQL backup integrity is essential to any reliable data protection strategy.

## Checking a mysqldump File for Corruption

A logical dump ends with a specific footer. If this footer is missing, the dump was interrupted or corrupted:

```bash
tail -5 full_backup.sql
```

A healthy dump ends with lines like:

```text
-- Dump completed on 2024-03-15 02:30:45
```

If this line is missing, the dump did not complete successfully.

## Verifying File Checksums

Generate a checksum when creating a backup, then verify it later:

```bash
# At backup time
sha256sum full_backup.sql > full_backup.sql.sha256

# At verification time
sha256sum -c full_backup.sql.sha256
```

For compressed backups:

```bash
mysqldump -u root -p mydb | gzip > mydb_backup.sql.gz
sha256sum mydb_backup.sql.gz > mydb_backup.sql.gz.sha256

# Verify integrity
sha256sum -c mydb_backup.sql.gz.sha256

# Verify the gzip file is not corrupted
gzip -t mydb_backup.sql.gz && echo "Archive is valid"
```

## Performing a Test Restore

The only definitive test is a full restore to a separate instance:

```bash
# Start a test MySQL instance (using Docker)
docker run -d --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=testpass \
  -p 3307:3306 \
  mysql:8.0

# Wait for it to start
sleep 10

# Restore the backup
mysql -h 127.0.0.1 -P 3307 -u root -ptestpass < full_backup.sql

echo "Restore completed successfully"
```

## Validating Row Counts After Restore

After restoring, compare row counts between the original and restored databases:

```sql
-- Run on original database
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY table_name;
```

```sql
-- Run on restored database (should match)
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY table_name;
```

For exact counts, query critical tables directly:

```sql
SELECT
  (SELECT COUNT(*) FROM mydb.orders) AS original_count,
  (SELECT COUNT(*) FROM mydb_restored.orders) AS restored_count;
```

## Using mysqlcheck to Validate Tables

After a restore, run `mysqlcheck` to verify table integrity:

```bash
mysqlcheck -u root -p --all-databases --check
```

For a specific database:

```bash
mysqlcheck -u root -p mydb --check --extended
```

## Validating XtraBackup Integrity

For physical backups created with Percona XtraBackup, use the `--prepare` step to validate:

```bash
xtrabackup --prepare --target-dir=/var/backup/mysql/latest

echo "Backup preparation (validation) exit code: $?"
```

A zero exit code confirms the backup is consistent and restorable.

## Automating Backup Testing

A weekly cron job that validates backup integrity:

```bash
#!/bin/bash
BACKUP_FILE=/var/backups/mysql/latest.sql.gz
LOG=/var/log/mysql-backup-test.log

echo "$(date): Starting backup validation" >> $LOG

# Check file exists and is non-empty
if [ ! -s "$BACKUP_FILE" ]; then
  echo "$(date): ERROR - Backup file missing or empty" >> $LOG
  exit 1
fi

# Verify gzip integrity
if ! gzip -t "$BACKUP_FILE" 2>/dev/null; then
  echo "$(date): ERROR - Backup file corrupted" >> $LOG
  exit 1
fi

# Check for dump completion marker
if ! zcat "$BACKUP_FILE" | tail -5 | grep -q "Dump completed"; then
  echo "$(date): ERROR - Dump did not complete" >> $LOG
  exit 1
fi

echo "$(date): Backup validation passed" >> $LOG
```

## Summary

Testing backup integrity requires more than just checking if a file exists. Verify checksums to detect file corruption, confirm dump completion markers in logical backups, and periodically perform full test restores to a separate instance. Automated weekly validation scripts catch issues before a real disaster forces your hand.
