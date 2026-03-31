# What Is the mysql_secure_installation Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Installation, Hardening

Description: mysql_secure_installation is an interactive script that hardens a fresh MySQL installation by setting the root password, removing test databases, and disabling insecure defaults.

---

## Overview

`mysql_secure_installation` is a shell script included with every MySQL installation that walks through essential security hardening steps for a new MySQL server. Running it immediately after installation is a security best practice before the server is put into production.

It addresses the most common security weaknesses in a default MySQL installation: anonymous users, remote root login, and the default test database.

## Running mysql_secure_installation

```bash
mysql_secure_installation
```

On systems with sudo:

```bash
sudo mysql_secure_installation
```

## What the Script Does

The script presents a series of interactive prompts:

### 1. Install/Validate Password Plugin

```text
Would you like to setup VALIDATE PASSWORD component?
Press y|Y for Yes, any other key for No: y

There are three levels of password validation policy:
LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special chars and dictionary file

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2
```

### 2. Set the Root Password

```text
Please set the password for root here.
New password: ************
Re-enter new password: ************
```

### 3. Remove Anonymous Users

```text
By default, a MySQL installation has an anonymous user, allowing anyone
to log into MySQL without having to have a user account created for them.
Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
```

### 4. Disallow Remote Root Login

```text
Normally, root should only be allowed to connect from 'localhost'.
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
```

### 5. Remove Test Database

```text
By default, MySQL comes with a database named 'test' that anyone can
access. Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
```

### 6. Reload Privilege Tables

```text
Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
```

## Non-Interactive (Automated) Execution

For automated deployments and scripts:

```bash
# Pass root password and auto-answer yes to all prompts
mysql_secure_installation \
  --password=your_root_password \
  --use-default \
  --host=127.0.0.1
```

Or using expect:

```bash
expect -c "
spawn mysql_secure_installation
expect \"Enter password for user root:\"
send \"root_password\r\"
expect \"New password:\"
send \"new_secure_password\r\"
expect \"Re-enter new password:\"
send \"new_secure_password\r\"
expect \"Remove anonymous users?\"
send \"y\r\"
expect \"Disallow root login remotely?\"
send \"y\r\"
expect \"Remove test database and access to it?\"
send \"y\r\"
expect \"Reload privilege tables now?\"
send \"y\r\"
expect eof
"
```

## What mysql_secure_installation Accomplishes (SQL Equivalent)

You can replicate what the script does manually:

```sql
-- Set root password
ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword123!';

-- Remove anonymous users
DELETE FROM mysql.user WHERE User = '';

-- Restrict root to localhost
DELETE FROM mysql.user WHERE User = 'root' AND Host != 'localhost';

-- Drop test database
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.db WHERE Db = 'test' OR Db = 'test\\_%';

-- Reload privileges
FLUSH PRIVILEGES;
```

## Additional Hardening After mysql_secure_installation

```sql
-- Create application user (never use root for applications)
CREATE USER 'app_user'@'10.0.0.%'
  IDENTIFIED WITH caching_sha2_password
  BY 'AppPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app_user'@'10.0.0.%';

-- Enable the validate_password component
INSTALL COMPONENT 'file://component_validate_password';
```

## Summary

`mysql_secure_installation` is a one-time setup script that should be run immediately after every new MySQL installation. It removes insecure defaults - anonymous users, test databases, and remote root access - that exist in a fresh install and sets up password validation. It takes less than a minute to run and dramatically reduces the attack surface of a new MySQL server.
