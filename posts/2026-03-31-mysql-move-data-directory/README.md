# How to Move the MySQL Data Directory to a New Location

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Directory, Storage, Migration, Administration

Description: Learn the step-by-step process to safely move the MySQL data directory to a new disk or path without data loss on Linux systems.

---

Moving the MySQL data directory is a common task when the original disk runs low on space or when you want to store database files on a faster storage volume. The process requires stopping MySQL, copying data with preserved permissions, updating configuration, and handling OS-level security policies.

## Step 1 - Verify Current Configuration

Before making any changes, record the current data directory:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Also check available disk space on the target location:

```bash
df -h /new/path
```

## Step 2 - Stop MySQL

Stop the MySQL service cleanly to avoid data corruption:

```bash
sudo systemctl stop mysql
```

Verify MySQL has stopped completely:

```bash
sudo systemctl status mysql
```

## Step 3 - Copy the Data Directory

Use `rsync` with archive mode to preserve permissions, ownership, and symbolic links:

```bash
sudo rsync -av /var/lib/mysql/ /new/mysql/
```

Do not use `cp -r` as it may not preserve all necessary file attributes. The trailing slash in the source path tells rsync to copy the contents, not the directory itself.

## Step 4 - Rename the Old Directory as a Backup

```bash
sudo mv /var/lib/mysql /var/lib/mysql.bak
```

Keep this backup until you have verified the new location works correctly.

## Step 5 - Update my.cnf

Edit the MySQL configuration file to point to the new location:

```ini
[mysqld]
datadir = /new/mysql
```

## Step 6 - Handle AppArmor (Ubuntu/Debian)

On Ubuntu, AppArmor restricts which paths MySQL can access. Edit the AppArmor alias file:

```bash
sudo nano /etc/apparmor.d/tunables/alias
```

Add the alias:

```text
alias /var/lib/mysql/ -> /new/mysql/,
```

Reload AppArmor:

```bash
sudo systemctl restart apparmor
```

Alternatively, add the new path directly to `/etc/apparmor.d/usr.sbin.mysqld`:

```text
/new/mysql/ r,
/new/mysql/** rwk,
```

## Step 7 - Handle SELinux (RHEL/CentOS/Rocky)

On RHEL-based systems, update the SELinux file context:

```bash
sudo semanage fcontext -a -t mysqld_db_t "/new/mysql(/.*)?"
sudo restorecon -Rv /new/mysql
```

## Step 8 - Start MySQL and Verify

Start MySQL and check the data directory:

```bash
sudo systemctl start mysql
```

```sql
SHOW VARIABLES LIKE 'datadir';
SHOW DATABASES;
```

Run a quick sanity check on your most important databases:

```bash
mysqlcheck -u root -p --all-databases
```

## Step 9 - Remove the Backup

Once you are confident the migration succeeded:

```bash
sudo rm -rf /var/lib/mysql.bak
```

## Summary

Moving the MySQL data directory is a straightforward but careful procedure. The key steps are stopping MySQL cleanly, copying with `rsync -a`, updating `my.cnf`, and handling OS security policies with AppArmor or SELinux. Always keep the old directory as a backup until the new location is verified working, and test all databases before removing the backup.
