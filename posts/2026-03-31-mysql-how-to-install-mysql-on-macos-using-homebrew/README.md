# How to Install MySQL on macOS Using Homebrew

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Macos, Homebrew, Installation, Database Setup

Description: Install MySQL on macOS using Homebrew with a few commands, then configure the root account and start the service for local development.

---

## Prerequisites

Before installing MySQL, make sure Homebrew is installed on your Mac. If not, install it by running:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Verify Homebrew is working:

```bash
brew --version
```

## Installing MySQL

Update Homebrew and then install MySQL:

```bash
brew update
brew install mysql
```

Homebrew will install the latest stable MySQL version. The installation output will show where the data directory and configuration file are located.

To install a specific version, use a versioned formula:

```bash
brew install mysql@8.0
```

## Starting the MySQL Service

After installation, start MySQL as a background service managed by Homebrew:

```bash
brew services start mysql
```

To start it only for the current session (not as a persistent service):

```bash
mysql.server start
```

Check that MySQL is running:

```bash
brew services list
```

You should see `mysql` with a status of `started`.

## Running the Secure Installation Script

MySQL ships with a security hardening script. Run it immediately after installation:

```bash
mysql_secure_installation
```

The script will prompt you to:
- Set a root password (if not already set)
- Remove anonymous users
- Disallow remote root login
- Remove the test database
- Reload privilege tables

Respond `Y` to each prompt for a secure configuration.

## Connecting to MySQL

After installation, connect to the MySQL shell as root:

```bash
mysql -u root -p
```

Enter the root password you set during `mysql_secure_installation`. You should see the MySQL prompt:

```text
mysql>
```

## Verifying the Installation

Check the MySQL version from within the shell:

```sql
SELECT VERSION();
```

Example output:

```text
+-----------+
| VERSION() |
+-----------+
| 8.0.37    |
+-----------+
```

Alternatively, check from the terminal:

```bash
mysql --version
```

## Stopping and Restarting MySQL

Stop the service:

```bash
brew services stop mysql
```

Restart the service:

```bash
brew services restart mysql
```

## Finding the Configuration File

Homebrew installs MySQL configuration under:

```text
/opt/homebrew/etc/my.cnf        (Apple Silicon Macs)
/usr/local/etc/my.cnf           (Intel Macs)
```

If the file does not exist, create it:

```bash
touch /opt/homebrew/etc/my.cnf
```

A minimal configuration example:

```text
[mysqld]
bind-address = 127.0.0.1
max_connections = 100
innodb_buffer_pool_size = 256M
```

## Uninstalling MySQL

To completely remove MySQL installed via Homebrew:

```bash
brew services stop mysql
brew uninstall mysql
rm -rf /opt/homebrew/var/mysql
rm -f /opt/homebrew/etc/my.cnf
```

This removes the binaries, data directory, and configuration file.

## Summary

Installing MySQL on macOS via Homebrew is straightforward: run `brew install mysql`, start the service with `brew services start mysql`, and secure the installation with `mysql_secure_installation`. Homebrew manages the MySQL service lifecycle, making it easy to start, stop, and upgrade MySQL for local development.
