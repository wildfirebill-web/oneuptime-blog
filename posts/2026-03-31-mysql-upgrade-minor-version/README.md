# How to Upgrade MySQL from One Minor Version to Another

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Upgrade, Administration, Version, Migration

Description: Learn the safe step-by-step process to upgrade MySQL from one minor version to another, such as from 8.0.32 to 8.0.36, with minimal downtime.

---

MySQL minor version upgrades - for example from 8.0.32 to 8.0.36 - are generally safe in-place upgrades that do not require data migration or schema changes. MySQL 8.0 supports automatic in-place upgrade through the package manager, but following a structured process protects against unexpected issues.

## Before You Begin

Always read the release notes for every version between your current and target version. Minor releases sometimes include behavior changes that affect applications.

Check your current version:

```sql
SELECT VERSION();
```

Or from the shell:

```bash
mysqld --version
```

## Step 1 - Back Up Your Data

Create a full backup before any upgrade:

```bash
mysqldump -u root -p --all-databases --single-transaction \
  --routines --triggers --events \
  > /backup/pre_upgrade_$(date +%Y%m%d).sql
```

For a physical backup using Percona XtraBackup:

```bash
xtrabackup --backup --target-dir=/backup/pre_upgrade --user=root --password=secret
```

## Step 2 - Check Table Compatibility

Run `mysqlcheck` to identify any tables that need attention:

```bash
mysqlcheck -u root -p --check-upgrade --all-databases
```

Fix any reported issues before proceeding.

## Step 3 - Stop the Application

Put your application in maintenance mode and verify no active connections exist:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

## Step 4 - Perform the Upgrade

On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install mysql-server
```

On RHEL/CentOS/Rocky Linux:

```bash
sudo yum update mysql-server
# or with dnf:
sudo dnf update mysql-server
```

On macOS with Homebrew:

```bash
brew upgrade mysql
```

The package manager replaces the MySQL binaries while preserving the data directory.

## Step 5 - Start MySQL and Run Upgrade

MySQL 8.0 performs automatic in-place upgrade on first startup after a version change:

```bash
sudo systemctl start mysql
```

Watch the error log for upgrade messages:

```bash
sudo tail -f /var/log/mysql/error.log
```

You should see lines like:

```text
[System] [MY-013381] [Server] Server upgrade from '80032' to '80036' started.
[System] [MY-013381] [Server] Server upgrade from '80032' to '80036' completed.
```

## Step 6 - Verify the Upgrade

Confirm the new version:

```sql
SELECT VERSION();
```

Run the full table check again:

```bash
mysqlcheck -u root -p --check --all-databases
```

Verify replication (if applicable):

```sql
SHOW REPLICA STATUS\G
```

## Step 7 - Restore Application Traffic

Remove maintenance mode and monitor for errors in the application and MySQL error log.

## Rollback Plan

If the upgrade fails:
1. Stop MySQL
2. Restore the previous MySQL binary version
3. Restore data from the pre-upgrade backup
4. Start MySQL

Keep the pre-upgrade backup for at least one week after a successful upgrade.

## Summary

MySQL minor version upgrades are in-place operations handled by the package manager. The key steps are backup, `mysqlcheck --check-upgrade`, package update, restart, and verification. MySQL 8.0 automates the internal upgrade process on first startup, writing progress to the error log. Always test on a staging environment before upgrading production.
