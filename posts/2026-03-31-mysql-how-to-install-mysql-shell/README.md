# How to Install MySQL Shell

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQL Shell, Installation, Administration

Description: Learn how to install MySQL Shell on Linux, macOS, and Windows, and explore its key features including JavaScript, Python, and SQL modes.

---

## What is MySQL Shell

MySQL Shell is an advanced command-line client and code editor for MySQL. Unlike the traditional `mysql` client, it supports three execution modes:

- **SQL mode** - standard SQL execution
- **JavaScript mode** - scripting with the X DevAPI
- **Python mode** - scripting with the X DevAPI

MySQL Shell also powers `mysqlsh` utilities like `dumpInstance`, `loadDump`, and `checkForServerUpgrade`.

## Installation on Ubuntu/Debian

```bash
# Add the MySQL APT repository
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb

# Update and install
sudo apt update
sudo apt install mysql-shell

# Verify
mysqlsh --version
```

## Installation on RHEL/CentOS/Fedora

```bash
# Add the MySQL YUM repository
sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

# Install MySQL Shell
sudo dnf install mysql-shell

# Verify
mysqlsh --version
```

## Installation on macOS

Using Homebrew:

```bash
brew install mysql-shell

# Verify
mysqlsh --version
```

Or download the macOS DMG from the MySQL downloads page and install manually.

## Installation on Windows

Download the MySQL Shell MSI installer from:

```text
https://dev.mysql.com/downloads/shell/
```

Run the installer and follow the wizard. After installation, `mysqlsh` is available in the Start menu and from the command prompt.

## Connecting to MySQL

```bash
# Connect with URI string
mysqlsh user@localhost:3306/mydb

# Connect interactively
mysqlsh --host=localhost --port=3306 --user=root --password

# Connect using classic MySQL protocol (not X Protocol)
mysqlsh --mysql --host=localhost --user=root
```

## Switching Modes

Once inside MySQL Shell:

```javascript
// Switch to SQL mode
\sql

// Switch to JavaScript mode
\js

// Switch to Python mode
\py

// Show current mode
\status
```

## Running SQL Queries

```sql
-- In \sql mode
\sql
SELECT VERSION();
SHOW DATABASES;
USE mydb;
SELECT * FROM employees LIMIT 10;
```

## Using JavaScript Mode

```javascript
\js
var session = mysqlx.getSession('root@localhost');
var db = session.getSchema('mydb');
var employees = db.getTable('employees');
var result = employees.select(['name', 'salary'])
  .where('salary > 80000')
  .execute();
print(result.fetchAll());
```

## Using Python Mode

```python
\py
session = mysqlx.get_session('root@localhost')
db = session.get_schema('mydb')
employees = db.get_table('employees')
result = employees.select(['name', 'salary']).where('salary > 80000').execute()
for row in result.fetch_all():
    print(row)
```

## Running Backup and Restore with MySQL Shell

```bash
# Dump a database (faster than mysqldump for large databases)
mysqlsh root@localhost -- util dumpSchemas mydb --outputUrl=/backups/mydb_dump

# Load a dump
mysqlsh root@localhost -- util loadDump /backups/mydb_dump
```

## Running Upgrade Compatibility Check

```bash
mysqlsh root@localhost -- util checkForServerUpgrade
```

## Summary

MySQL Shell is installed via package managers on Linux, Homebrew on macOS, or the MSI installer on Windows. It extends the classic MySQL client with JavaScript/Python scripting, X DevAPI support, and powerful utilities for backup, restore, and upgrade checks. Use `\sql`, `\js`, and `\py` to switch between execution modes interactively.
