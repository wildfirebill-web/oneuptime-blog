# How to Fix ERROR 1698 Access Denied for User root@localhost in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Root, Authentication, Security

Description: Fix MySQL ERROR 1698 Access denied for user root@localhost by switching from auth_socket to mysql_native_password or setting a proper root password.

---

MySQL ERROR 1698 specifically affects the root user on Ubuntu and Debian systems. The error reads: `ERROR 1698 (28000): Access denied for user 'root'@'localhost'`. Even when providing the correct password, the connection is refused. This happens because the root account uses the `auth_socket` plugin by default on these distributions.

## Why This Happens

On Ubuntu 16.04+ and Debian, MySQL's root account is configured to authenticate via the operating system's socket authentication (`auth_socket` plugin) rather than a password. This means only the `root` OS user can connect as MySQL root without a password.

## Check the Authentication Plugin

```sql
-- Connect as the root OS user first
sudo mysql

-- Check the plugin for root
SELECT User, Host, plugin, authentication_string
FROM mysql.user
WHERE User = 'root';
```

If `plugin` shows `auth_socket` or `unix_socket`, that is the cause.

## Fix 1: Connect Using sudo

The auth_socket plugin allows the OS root user to connect without a password:

```bash
# Connect without a password as the OS root user
sudo mysql

# Or
sudo mysql -u root
```

This is the intended behavior on Ubuntu. No password is required when using `sudo`.

## Fix 2: Change root to Password Authentication

If you need to connect from applications or tools that cannot use `sudo`:

```bash
sudo mysql
```

```sql
-- For MySQL 8.0+
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'your_new_password';

-- For MySQL 5.7 (better compatibility with older clients)
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_new_password';

FLUSH PRIVILEGES;
```

Now test the connection with a password:

```bash
mysql -u root -p
```

## Fix 3: Reset Root Password via Safe Mode

If you do not know the current root password and cannot use `sudo mysql`:

```bash
sudo systemctl stop mysql
sudo mysqld_safe --skip-grant-tables &

# Connect without a password
mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new_password';
FLUSH PRIVILEGES;
QUIT;
```

```bash
sudo kill $(cat /var/run/mysqld/mysqld.pid)
sudo systemctl start mysql
mysql -u root -p
```

## Fix 4: Create a Separate Admin User

Instead of changing root, create a new admin user with password authentication:

```bash
sudo mysql
```

```sql
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

```bash
mysql -u admin -p
```

## Fix 5: Update my.cnf Connection Configuration

After changing authentication, update any application configuration:

```text
[client]
user = root
password = your_new_password
host = localhost
```

## Summary

ERROR 1698 on Ubuntu/Debian is caused by the `auth_socket` plugin, which restricts root MySQL access to the OS root user. The fix is either using `sudo mysql` for interactive access, changing the authentication plugin to `mysql_native_password` with `ALTER USER`, or creating a separate admin user with password authentication for programmatic access. Never disable `auth_socket` without setting a strong root password first.
