# How to Uninstall MySQL Completely on Ubuntu

Author: [OneUptime](https://oneuptime.com)

Tags: MySQL, Uninstall, Ubuntu, Linux, Administration

Description: Completely remove MySQL from Ubuntu by stopping the service, purging all packages, deleting data directories, and cleaning up residual configuration files.

---

## How It Works

Uninstalling MySQL on Ubuntu requires more than just `apt remove`. The `--purge` flag removes configuration files, but you must also manually delete the data directory and any leftover files in `/etc`, `/var`, and `/run` to achieve a completely clean state.

```mermaid
flowchart LR
    A[Stop MySQL service] --> B[apt purge mysql-*]
    B --> C[apt autoremove]
    C --> D[Delete /var/lib/mysql]
    D --> E[Delete /etc/mysql]
    E --> F[Delete residual log files]
    F --> G[Clean APT repository config]
    G --> H[Verify removal]
```

## Step 1 - Stop the MySQL Service

```bash
sudo systemctl stop mysql
sudo systemctl disable mysql
```

## Step 2 - Uninstall MySQL Packages

Remove all MySQL packages including configuration files with `--purge`.

```bash
sudo apt purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-* -y
```

If you used the official MySQL APT repository:

```bash
sudo apt purge mysql-server mysql-client mysql-community-server mysql-community-client \
    mysql-community-common mysql-community-libs mysql-apt-config -y
```

## Step 3 - Remove Dependencies

```bash
sudo apt autoremove -y
sudo apt autoclean
```

## Step 4 - Delete the Data Directory

The data directory contains all databases and is not removed by `apt purge`.

```bash
sudo rm -rf /var/lib/mysql
sudo rm -rf /var/lib/mysql-files
sudo rm -rf /var/lib/mysql-keyring
```

## Step 5 - Delete Configuration Files

```bash
sudo rm -rf /etc/mysql
```

Check for any remaining MySQL files in `/etc`.

```bash
sudo find /etc -name "mysql*" 2>/dev/null
```

## Step 6 - Remove Log Files

```bash
sudo rm -rf /var/log/mysql
```

## Step 7 - Remove the MySQL System User and Group

```bash
sudo deluser mysql
sudo delgroup mysql
```

## Step 8 - Remove the MySQL APT Repository (If Added)

If you installed the official MySQL APT configuration package:

```bash
sudo rm -f /etc/apt/sources.list.d/mysql.list
sudo rm -f /etc/apt/trusted.gpg.d/mysql.gpg
sudo apt update
```

## Step 9 - Remove PID and Socket Files

```bash
sudo rm -f /var/run/mysqld/mysqld.pid
sudo rm -f /var/run/mysqld/mysqld.sock
sudo rm -rf /var/run/mysqld
```

## Step 10 - Verify Complete Removal

Check that no MySQL processes are running.

```bash
ps aux | grep -i mysql
```

Check that no MySQL packages are installed.

```bash
dpkg -l | grep -i mysql
```

Both commands should return no results (or only the grep process itself).

Check that port 3306 is not in use.

```bash
sudo ss -tlnp | grep 3306
```

No output means MySQL is not listening.

## Backup Before Uninstalling

Before uninstalling, create a backup of your databases if you need the data later.

```bash
# Backup all databases
mysqldump -u root -p --all-databases > /backup/all-databases-$(date +%Y%m%d).sql

# Backup a specific database
mysqldump -u root -p myapp > /backup/myapp-$(date +%Y%m%d).sql
```

## Reinstalling MySQL After Removal

After a complete removal, you can reinstall MySQL fresh.

```bash
sudo apt update
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

## Summary

Completely uninstalling MySQL on Ubuntu requires stopping the service, purging all packages with `apt purge`, deleting the data directory at `/var/lib/mysql`, removing `/etc/mysql`, clearing log files, and removing the MySQL system user. The key distinction from a regular `apt remove` is the `--purge` flag plus the manual deletion of data and configuration directories that the package manager does not touch.
