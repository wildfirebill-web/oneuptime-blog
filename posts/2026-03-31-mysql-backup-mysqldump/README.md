# How to Back Up MySQL with mysqldump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, mysqldump, Database Administration

Description: Learn how to use mysqldump to create logical backups of MySQL databases, including full, partial, and consistent backup strategies with scheduling examples.

---

## How mysqldump Works

`mysqldump` is MySQL's built-in logical backup tool. It connects to the MySQL server, reads table structures and data, and writes them as SQL statements (CREATE TABLE, INSERT, etc.) to a text file. To restore, you simply replay the SQL file against a MySQL server.

Advantages:
- Backups are human-readable SQL files
- Easy to restore a single table or database
- Portable across MySQL versions and platforms
- No server downtime required

Disadvantages:
- Slower than physical backups for large databases
- Requires locking (FLUSH TABLES WITH READ LOCK) for consistent non-InnoDB backups
- Restore time is proportional to data size

## Basic Usage

### Back Up a Single Database

```bash
mysqldump -u root -p myapp_db > /backups/myapp_db_$(date +%Y%m%d).sql
```

### Back Up Multiple Databases

```bash
mysqldump -u root -p --databases myapp_db analytics_db > /backups/multi_db_backup.sql
```

### Back Up All Databases

```bash
mysqldump -u root -p --all-databases > /backups/all_databases_$(date +%Y%m%d).sql
```

### Back Up a Single Table

```bash
mysqldump -u root -p myapp_db orders > /backups/orders_$(date +%Y%m%d).sql
```

## Consistent InnoDB Backup

For InnoDB tables, use `--single-transaction` to take a consistent snapshot without locking tables:

```bash
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    myapp_db > /backups/myapp_db_consistent.sql
```

`--single-transaction` starts a transaction before the dump and reads all data within that transaction, ensuring a consistent snapshot without blocking writes.

## Including Stored Routines and Events

By default, mysqldump does not include stored procedures, functions, or events. Add these flags:

```bash
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --all-databases > /backups/full_backup.sql
```

## Compressed Backups

Pipe the output through gzip to reduce file size:

```bash
mysqldump -u root -p \
    --single-transaction \
    --all-databases | gzip > /backups/full_backup_$(date +%Y%m%d).sql.gz
```

To view a compressed backup without extracting:

```bash
zcat /backups/full_backup_20260331.sql.gz | head -50
```

## Remote Backup

Connect to a remote MySQL server:

```bash
mysqldump -u root -p \
    -h db-server.example.com \
    --single-transaction \
    myapp_db > /backups/remote_myapp_db.sql
```

## Using a Credentials File

Avoid putting passwords on the command line by using a credentials file:

```ini
# /etc/mysql/backup.cnf (chmod 600)
[client]
user     = backup_user
password = BackupPass123!
host     = localhost
```

Then use:

```bash
mysqldump --defaults-extra-file=/etc/mysql/backup.cnf \
    --single-transaction \
    myapp_db > /backups/myapp_db.sql
```

## Creating a Backup User

Create a dedicated backup user with minimal privileges:

```sql
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'BackupPass123!';
GRANT SELECT, SHOW VIEW, RELOAD, REPLICATION CLIENT, EVENT, LOCK TABLES, TRIGGER
    ON *.* TO 'backup_user'@'localhost';
FLUSH PRIVILEGES;
```

## Automating Backups with cron

Create a backup script at `/usr/local/bin/mysql_backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
CREDS="--defaults-extra-file=/etc/mysql/backup.cnf"

mkdir -p "$BACKUP_DIR"

# Full backup
mysqldump $CREDS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --all-databases | gzip > "$BACKUP_DIR/full_$DATE.sql.gz"

# Remove backups older than retention days
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $BACKUP_DIR/full_$DATE.sql.gz"
```

Make it executable and schedule it:

```bash
chmod +x /usr/local/bin/mysql_backup.sh

# Add to crontab - run daily at 2 AM
crontab -e
```

```text
0 2 * * * /usr/local/bin/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1
```

## Verifying a Backup

Test that the backup file is valid:

```bash
# Check file is not empty and has content
ls -lh /backups/myapp_db_20260331.sql

# Check the SQL starts with expected comments
head -20 /backups/myapp_db_20260331.sql

# Count number of INSERT statements
grep -c '^INSERT' /backups/myapp_db_20260331.sql
```

## Best Practices

- Always use `--single-transaction` for InnoDB databases to avoid locking.
- Include `--routines`, `--triggers`, and `--events` for complete backups.
- Compress backups with gzip to reduce storage requirements.
- Store backups on a separate disk or remote storage (S3, NFS, etc.).
- Verify backups regularly by doing test restores to a staging server.
- Use a dedicated backup user with only the necessary privileges.
- Retain multiple generations of backups (daily for 7 days, weekly for 4 weeks).
- For large databases, consider Percona XtraBackup for faster physical backups.

## Summary

`mysqldump` creates portable, SQL-based logical backups of MySQL databases. Use `--single-transaction` for InnoDB tables to get consistent backups without downtime, and compress output with gzip to save space. Automate backups with cron and always test restores periodically to ensure backup integrity.
