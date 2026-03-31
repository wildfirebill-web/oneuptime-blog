# How to Use mysql_config_editor for Secure Login Paths

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Command Line, Authentication

Description: Learn how to use mysql_config_editor to store MySQL credentials securely in login paths, avoiding passwords in scripts and command-line history.

---

## What Is mysql_config_editor?

`mysql_config_editor` is a MySQL utility that stores connection credentials (host, port, user, password, socket) in an obfuscated file at `~/.mylogin.cnf`. Client programs like `mysql`, `mysqldump`, `mysqlimport`, and `mysqlshow` can then reference these stored credentials via a "login path" name instead of specifying passwords on the command line.

This prevents passwords from appearing in shell history, process lists (`ps aux`), and shell scripts.

## Basic Syntax

```bash
mysql_config_editor [command] [options]
```

Commands: `set`, `remove`, `print`, `reset`

## Creating a Login Path

```bash
# Create a login path called 'local' for a local MySQL instance
mysql_config_editor set \
  --login-path=local \
  --host=127.0.0.1 \
  --port=3306 \
  --user=root \
  --password
# You will be prompted for the password (not echoed)
```

```bash
# Create a login path for a remote production server
mysql_config_editor set \
  --login-path=prod \
  --host=db.prod.example.com \
  --port=3306 \
  --user=dba_user \
  --password
```

## Listing Stored Login Paths

```bash
# Print all stored login paths (passwords are shown obfuscated)
mysql_config_editor print --all
```

```text
[local]
user = root
password = *****
host = 127.0.0.1
port = 3306
[prod]
user = dba_user
password = *****
host = db.prod.example.com
port = 3306
```

## Using a Login Path with MySQL Tools

```bash
# Connect to MySQL using the 'local' login path
mysql --login-path=local

# Use with a specific database
mysql --login-path=local mydb

# Use with mysqldump
mysqldump --login-path=prod mydb orders > orders_backup.sql

# Use with mysqlimport
mysqlimport --login-path=local mydb /tmp/customers.csv

# Use with mysqlshow
mysqlshow --login-path=prod mydb
```

## Using a Socket Instead of TCP

```bash
mysql_config_editor set \
  --login-path=local_socket \
  --socket=/var/run/mysqld/mysqld.sock \
  --user=root \
  --password
```

## Default Login Path

If you name a login path `client`, it will be used automatically when no `--login-path` is specified:

```bash
mysql_config_editor set \
  --login-path=client \
  --host=127.0.0.1 \
  --user=root \
  --password

# Now this works without any connection flags
mysql
```

## Removing a Login Path

```bash
# Remove a specific login path
mysql_config_editor remove --login-path=prod

# Remove a specific option from a login path
mysql_config_editor remove --login-path=local --password

# Remove all login paths (reset the entire file)
mysql_config_editor reset
```

## File Location and Security

```bash
# The credentials file is stored at:
ls -la ~/.mylogin.cnf

# It should be readable only by the owner
chmod 600 ~/.mylogin.cnf
```

The file uses obfuscation (not strong encryption), so treat it like a password file - restrict access and never commit it to version control.

## Comparison: login-path vs Environment Variable

```bash
# Old insecure approach - password visible in process list
mysql -u root -pMyPassword

# Better approach - but still in shell history
export MYSQL_PWD=MyPassword
mysql -u root

# Best approach - credentials in ~/.mylogin.cnf
mysql --login-path=local
```

## Summary

`mysql_config_editor` is the recommended way to store MySQL credentials for command-line tools. By saving connection details to `~/.mylogin.cnf` and referencing them by login path name, you eliminate passwords from shell history, process lists, and scripts. Always set the file permissions to `600` and treat it with the same care as any secrets file.
