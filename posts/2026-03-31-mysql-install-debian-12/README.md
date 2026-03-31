# How to Install MySQL on Debian 12

Author: [OneUptime](https://oneuptime.com)

Tags: MySQL, Installation, Debian, Linux, Database

Description: Install MySQL 8.0 on Debian 12 Bookworm using the official MySQL APT repository, secure the server, and create a production-ready application user.

---

## How It Works

Debian 12 (Bookworm) does not include MySQL in its main repository; it ships MariaDB instead. The official MySQL APT repository from dev.mysql.com provides MySQL 8.0 and 8.4 packages that are compatible with Debian 12.

```mermaid
flowchart LR
    A[Download MySQL APT config package] --> B[dpkg -i mysql-apt-config.deb]
    B --> C[apt update]
    C --> D[apt install mysql-server]
    D --> E[Set root password during install]
    E --> F[mysql_secure_installation]
    F --> G[MySQL ready]
```

## Prerequisites

- Debian 12 Bookworm (fresh install or existing system)
- User with `sudo` privileges
- `wget` installed (`sudo apt install -y wget`)

## Step 1 - Download the MySQL APT Configuration Package

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
```

## Step 2 - Install the Configuration Package

```bash
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
```

A dialog box appears. Select **MySQL Server & Cluster** and then **mysql-8.0**, then select **OK**.

## Step 3 - Update the Package Index

```bash
sudo apt update
```

## Step 4 - Install MySQL Server

```bash
sudo apt install -y mysql-server
```

During installation, you are prompted to set a root password. Enter and confirm a strong password. The authentication plugin selection screen appears next - choose **Use Strong Password Encryption (RECOMMENDED)** (caching_sha2_password).

## Step 5 - Verify the Service

```bash
sudo systemctl status mysql
```

```text
● mysql.service - MySQL Community Server
     Active: active (running)
```

## Step 6 - Run the Security Script

```bash
sudo mysql_secure_installation
```

Respond to each prompt:

```text
Change root password?            N (already set during install)
Remove anonymous users?          Y
Disallow root login remotely?    Y
Remove test database?            Y
Reload privilege tables?         Y
```

## Step 7 - Connect to MySQL

```bash
mysql -u root -p
```

Create an application database and user.

```sql
CREATE DATABASE webapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'webapp_user'@'localhost' IDENTIFIED BY 'Secure$Pass1';
GRANT ALL PRIVILEGES ON webapp.* TO 'webapp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Key File Locations

```text
/etc/mysql/mysql.conf.d/mysqld.cnf   Server configuration
/etc/mysql/conf.d/                   Drop-in configuration directory
/var/lib/mysql/                       Data directory
/var/log/mysql/error.log             Error log
```

## Verify the Installation

```bash
mysql --version
```

```text
mysql  Ver 8.0.x  Distrib 8.0.x, for debian-linux-gnu (x86_64)
```

Check listening port.

```bash
sudo ss -tlnp | grep 3306
```

```text
LISTEN   0   151   127.0.0.1:3306   0.0.0.0:*   users:(("mysqld",...))
```

## Firewall Configuration (UFW)

If UFW is enabled, allow MySQL from specific hosts.

```bash
sudo ufw allow from 192.168.1.0/24 to any port 3306
sudo ufw reload
```

## Summary

Installing MySQL on Debian 12 requires adding the official MySQL APT repository because Debian's default repos only provide MariaDB. The `mysql-apt-config` package configures the repository interactively. During installation you set the root password and select the authentication plugin. Run `mysql_secure_installation` afterward to remove test accounts and harden the default setup.
