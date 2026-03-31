# How to Use MySQL Enterprise Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Enterprise Backup, MEB, Backup And Recovery, MySQL Enterprise

Description: Learn how to use MySQL Enterprise Backup (MEB) to perform hot, online backups of MySQL databases with minimal downtime and efficient incremental strategies.

---

## What Is MySQL Enterprise Backup

MySQL Enterprise Backup (MEB) is a commercial backup tool included with MySQL Enterprise Edition. It provides hot backups - backups taken while MySQL is fully online and serving queries - with no table locks for InnoDB tables. MEB supports full backups, incremental backups, compressed backups, and partial backups of selected databases or tables.

Key advantages over `mysqldump`:
- Hot backup with no locking for InnoDB
- Much faster for large databases (physical file copy vs logical export)
- Incremental backups
- Consistent point-in-time snapshots

## Prerequisites

```bash
# Verify MEB is installed
mysqlbackup --version

# Required MySQL user privileges
GRANT SELECT, BACKUP_ADMIN, RELOAD, PROCESS, SUPER,
      REPLICATION CLIENT ON *.* TO 'meb_user'@'localhost';
```

## Full Backup

```bash
mysqlbackup \
    --user=meb_user \
    --password=secret \
    --host=localhost \
    --backup-dir=/backup/full_$(date +%Y%m%d) \
    backup-and-apply-log
```

The `backup-and-apply-log` command takes the backup and applies any pending InnoDB redo logs, making the backup immediately ready for restore.

A two-step alternative (useful for parallel processing):

```bash
# Step 1: Take the backup
mysqlbackup --user=meb_user --password=secret \
    --backup-dir=/backup/raw_full backup

# Step 2: Apply log
mysqlbackup --backup-dir=/backup/raw_full apply-log
```

## Backup Directory Structure

After a full backup:

```text
/backup/full_20260331/
  backup-my.cnf         - MySQL configuration snapshot
  meta/                 - Backup metadata
  datadir/              - Copy of InnoDB data files
    ibdata1
    undo_001
    mydb/
      orders.ibd
      customers.ibd
  binlog/               - Binary log files at backup time
```

## Incremental Backup

Incremental backups copy only changed data pages since the last backup:

```bash
# Full backup first
mysqlbackup --user=meb_user --password=secret \
    --backup-dir=/backup/full_20260331 \
    backup-and-apply-log

# First incremental backup
mysqlbackup --user=meb_user --password=secret \
    --backup-dir=/backup/inc_20260401 \
    --incremental \
    --incremental-base=dir:/backup/full_20260331 \
    backup
```

## Compressed Backup

Save storage space with compression:

```bash
mysqlbackup --user=meb_user --password=secret \
    --backup-dir=/backup/compressed_20260331 \
    --compress \
    backup-and-apply-log
```

Compression ratio is typically 3:1 to 5:1 for typical database workloads.

## Restoring a Full Backup

```bash
# Stop MySQL
systemctl stop mysqld

# Clear the data directory
rm -rf /var/lib/mysql/*

# Restore
mysqlbackup \
    --defaults-file=/etc/my.cnf \
    --backup-dir=/backup/full_20260331 \
    copy-back

# Fix permissions
chown -R mysql:mysql /var/lib/mysql

# Start MySQL
systemctl start mysqld
```

## Restoring an Incremental Backup

Apply incremental backups to the full backup before restoring:

```bash
# Apply full backup log
mysqlbackup --backup-dir=/backup/full_20260331 apply-log

# Apply incremental to the full backup
mysqlbackup \
    --backup-dir=/backup/full_20260331 \
    --incremental-backup-dir=/backup/inc_20260401 \
    apply-incremental-backup

# Restore the merged backup
mysqlbackup \
    --defaults-file=/etc/my.cnf \
    --backup-dir=/backup/full_20260331 \
    copy-back
```

## Validating a Backup

Verify backup integrity without restoring:

```bash
mysqlbackup --backup-dir=/backup/full_20260331 validate
```

## Point-in-Time Recovery with Binary Logs

After restoring the backup, replay binary logs to recover to a specific point:

```bash
# Find the binary log position from the backup
cat /backup/full_20260331/meta/backup_variables.txt
# Look for: binlog_position

# Apply binary logs after backup's log position
mysqlbinlog \
    --start-position=4523891 \
    --stop-datetime="2026-03-31 14:30:00" \
    /var/lib/mysql/mysql-bin.000010 \
    | mysql -u root -p
```

## Summary

MySQL Enterprise Backup provides fast, hot, non-blocking backups of MySQL databases with support for full, incremental, and compressed backup strategies. Its physical file-copy approach is significantly faster than logical exports for large databases. Combine MEB with binary log replay for point-in-time recovery to any moment after the last backup.
