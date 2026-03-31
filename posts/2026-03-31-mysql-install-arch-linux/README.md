# How to Install MySQL on Arch Linux

Author: [OneUptime](https://oneuptime.com)

Tags: MySQL, Installation, Arch Linux, Linux, Database

Description: Install MySQL on Arch Linux using the official community repository via pacman, initialize the data directory, and configure the systemd service.

---

## How It Works

Arch Linux includes MySQL (the Oracle community build) in its `extra` repository. Unlike Debian or RHEL-based systems, Arch requires a manual data directory initialization step after installing the package before starting the service.

```mermaid
flowchart LR
    A[pacman -S mysql] --> B[mysqld --initialize --user=mysql]
    B --> C[systemctl start mysqld]
    C --> D[Read temp password from log]
    D --> E[mysql_secure_installation]
    E --> F[MySQL ready]
```

## Prerequisites

- Up-to-date Arch Linux installation (`sudo pacman -Syu`)
- User with `sudo` access
- Internet connection for package downloads

## Step 1 - Update the System

Always sync the package database before installing on Arch.

```bash
sudo pacman -Syu
```

## Step 2 - Install MySQL

```bash
sudo pacman -S mysql
```

At the prompt, confirm the installation. Pacman installs the MySQL server and client binaries.

Note: If you prefer MariaDB (which is the Arch community's recommended MySQL drop-in), install `mariadb` instead. This guide focuses on the Oracle MySQL package.

## Step 3 - Initialize the Data Directory

Arch Linux does not automatically initialize the MySQL data directory. You must do this manually.

```bash
sudo mysqld --initialize --user=mysql
```

This command:
- Creates `/var/lib/mysql` and sets ownership to the `mysql` user
- Initializes system tables
- Generates a temporary root password in the error log

Retrieve the temporary password.

```bash
sudo grep 'temporary password' /var/log/mysql/mysqld.log
```

```text
[Note] [MY-010454] [Server] A temporary password is generated for root@localhost: Temp#Pass1!
```

## Step 4 - Start and Enable the MySQL Service

```bash
sudo systemctl enable --now mysqld
```

Verify it is running.

```bash
sudo systemctl status mysqld
```

```text
● mysqld.service - MySQL Server
     Active: active (running)
```

## Step 5 - Secure the Installation

```bash
sudo mysql_secure_installation
```

Enter the temporary password when prompted. You must set a new root password before making other changes. Accept all hardening prompts.

## Step 6 - Connect and Create a User

```bash
mysql -u root -p
```

```sql
CREATE DATABASE devdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'dev'@'localhost' IDENTIFIED BY 'DevPwd1!Arch';
GRANT ALL PRIVILEGES ON devdb.* TO 'dev'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Verify the Installation

```bash
mysql --version
```

```text
mysql  Ver 8.0.x  Distrib 8.0.x, for Linux (x86_64)
```

## Key File Locations

```text
/etc/mysql/my.cnf             Primary configuration file
/var/lib/mysql/               Data directory
/var/log/mysql/mysqld.log    Error log
/run/mysqld/mysqld.pid       PID file
/run/mysqld/mysqld.sock      Unix socket
```

## Custom Configuration

Edit `/etc/mysql/my.cnf` to tune the server.

```ini
[mysqld]
character-set-server  = utf8mb4
collation-server      = utf8mb4_unicode_ci
innodb_buffer_pool_size = 512M
max_connections       = 200
slow_query_log        = 1
long_query_time       = 2
```

Restart after changes.

```bash
sudo systemctl restart mysqld
```

## Using MySQL with Arch Wiki Notes

Arch recommends MariaDB as the default MySQL implementation for Arch Linux due to better community packaging. However, if your application requires the Oracle MySQL feature set (e.g., MySQL-specific JSON functions, Group Replication), the `mysql` package from `extra` is the correct choice.

Regularly check for updates via the Arch User Repository (AUR) notes when MySQL upgrades are published.

```bash
sudo pacman -Syu mysql
```

## Summary

Installing MySQL on Arch Linux requires a manual `mysqld --initialize` step because Arch packages do not run post-install scripts that initialize the data directory automatically. After initialization, start the service with `systemctl`, retrieve the temporary root password from the error log, and run `mysql_secure_installation`. Arch's rolling-release model means MySQL updates arrive quickly; run `pacman -Syu` regularly to stay current.
