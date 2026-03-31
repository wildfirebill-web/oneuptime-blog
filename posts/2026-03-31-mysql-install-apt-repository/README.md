# How to Install MySQL Using the APT Repository

Author: [OneUptime](https://oneuptime.com)

Tags: MySQL, Installation, APT, Ubuntu, Debian

Description: Add the official MySQL APT repository to Ubuntu or Debian and install the latest MySQL server using apt, including GPG key verification and version pinning.

---

## How It Works

Oracle maintains an official APT repository at `repo.mysql.com` that provides MySQL 8.0, MySQL 8.4 LTS, MySQL 9.x, MySQL Shell, MySQL Router, and related tools for Debian and Ubuntu. Adding this repository gives you the most recent MySQL release and allows in-place minor version upgrades via `apt upgrade`.

```mermaid
flowchart LR
    A[Download mysql-apt-config .deb] --> B[dpkg -i mysql-apt-config]
    B --> C[apt update]
    C --> D[apt install mysql-server]
    D --> E[mysqld service starts]
    E --> F[mysql_secure_installation]
```

## Prerequisites

- Ubuntu 20.04, 22.04, or 24.04 OR Debian 11 or 12
- User with `sudo` privileges
- `wget` and `gnupg` installed

```bash
sudo apt install -y wget gnupg
```

## Step 1 - Download the APT Configuration Package

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
```

Verify the download integrity (optional but recommended).

```bash
md5sum mysql-apt-config_0.8.29-1_all.deb
```

## Step 2 - Install the Configuration Package

```bash
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
```

A dialog box appears asking which MySQL product series to enable.

```text
Which MySQL product do you wish to configure?
> MySQL Server & Cluster (Currently selected: mysql-8.0)
  MySQL Tools & Connectors (Currently selected: Enabled)
  MySQL Preview Packages (Currently selected: Disabled)
  Ok
```

Select **MySQL Server & Cluster**, choose the desired version (`mysql-8.0` or `mysql-8.4`), select **Ok**, then select **Ok** again to finish.

## Step 3 - Update the Package Index

```bash
sudo apt update
```

You should see MySQL repository entries being processed.

```text
Get:1 http://repo.mysql.com/apt/ubuntu jammy InRelease [21.5 kB]
Get:2 http://repo.mysql.com/apt/ubuntu jammy/mysql-8.0 amd64 Packages [10.6 kB]
```

## Step 4 - Install MySQL Server

```bash
sudo apt install -y mysql-server
```

During installation, a dialog prompts for the root password and authentication plugin selection. Choose **Use Strong Password Encryption** (caching_sha2_password).

## Step 5 - Verify the Installed Version

```bash
mysql --version
```

```text
mysql  Ver 8.0.x  Distrib 8.0.x, for Linux (x86_64) using  EditLine wrapper
```

Check the running service.

```bash
sudo systemctl status mysql
```

## Step 6 - Secure the Installation

```bash
sudo mysql_secure_installation
```

## Switching Between MySQL Versions

To change the active MySQL version (e.g., from 8.0 to 8.4), reconfigure the APT config package.

```bash
sudo dpkg-reconfigure mysql-apt-config
```

Select the new version, then update.

```bash
sudo apt update
sudo apt install -y mysql-server
```

## Pinning the MySQL Version

To prevent automatic major version upgrades, create an apt preferences file.

```bash
sudo tee /etc/apt/preferences.d/mysql << 'EOF'
Package: mysql-server
Pin: version 8.0.*
Pin-Priority: 1001
EOF
```

## Listing Available Packages from the MySQL Repo

```bash
apt-cache policy mysql-server
apt-cache show mysql-server
```

## Repository File Locations

After installation, the repository configuration is stored at:

```text
/etc/apt/sources.list.d/mysql.list
```

View the file contents.

```bash
cat /etc/apt/sources.list.d/mysql.list
```

```text
deb http://repo.mysql.com/apt/ubuntu/ jammy mysql-apt-config
deb http://repo.mysql.com/apt/ubuntu/ jammy mysql-8.0
deb http://repo.mysql.com/apt/ubuntu/ jammy mysql-tools
```

## Available Package Names

```bash
apt-cache search mysql-community
```

Common packages from the repo:

```text
mysql-community-server     Core server binary
mysql-community-client     Command-line client
mysql-shell                MySQL Shell (mysqlsh)
mysql-router               MySQL Router
mysql-workbench-community  MySQL Workbench GUI
```

## Summary

The official MySQL APT repository provides a controlled way to install and upgrade MySQL on Debian and Ubuntu systems. The `mysql-apt-config` package handles repository configuration and version selection. Once added, MySQL can be installed and updated using standard `apt` commands. Pin the version in `/etc/apt/preferences.d/` to prevent unintended major version upgrades during routine system updates.
