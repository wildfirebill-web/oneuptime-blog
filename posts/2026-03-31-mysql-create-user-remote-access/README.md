# How to Create a User for Remote Access in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Remote Access, Security, Database Administration

Description: Learn how to create a MySQL user account that can connect from a remote host, configure the server to accept remote connections, and secure remote access.

---

## Understanding MySQL Host-Based Authentication

Every MySQL account is a username-host pair. The host portion controls which IP addresses or hostnames are allowed to use that account. To allow remote connections, you must both configure the server to listen on a network interface and create users with appropriate host specifications.

## Step 1 - Allow MySQL to Listen on a Network Interface

By default, MySQL binds to `127.0.0.1` (localhost only). Edit the MySQL configuration:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change or set:

```text
bind-address = 0.0.0.0
```

Then restart MySQL:

```bash
sudo systemctl restart mysql
```

## Step 2 - Create the Remote User

Use `%` to allow connections from any host:

```sql
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'Str0ng!RemotePass2024';
```

Or restrict to a specific IP address:

```sql
CREATE USER 'remote_user'@'203.0.113.10' IDENTIFIED BY 'Str0ng!RemotePass2024';
```

Or restrict to a subnet:

```sql
CREATE USER 'remote_user'@'10.0.1.%' IDENTIFIED BY 'Str0ng!RemotePass2024';
```

## Step 3 - Grant Privileges

```sql
-- Typical application user for a specific database
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'remote_user'@'%';
```

## Step 4 - Open the Firewall Port

Allow incoming connections to MySQL's port (default 3306):

```bash
# UFW (Ubuntu)
sudo ufw allow from 203.0.113.10 to any port 3306

# firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.10" port protocol="tcp" port="3306" accept'
sudo firewall-cmd --reload
```

## Step 5 - Test the Remote Connection

From the remote client machine:

```bash
mysql -h YOUR_MYSQL_SERVER_IP -u remote_user -p mydb
```

If connection fails, check:

```bash
# Verify MySQL is listening on all interfaces
sudo ss -tlnp | grep 3306

# Verify the user exists
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user='remote_user';"
```

## Using a Specific Authentication Plugin

For older client compatibility:

```sql
CREATE USER 'legacy_remote'@'%'
  IDENTIFIED WITH mysql_native_password
  BY 'LegacyPass!';
```

## Revoking Remote Access

```sql
-- Drop the remote account entirely
DROP USER 'remote_user'@'%';

-- Or change to localhost-only by creating a new account and dropping the old one
CREATE USER 'remote_user'@'localhost' IDENTIFIED BY 'Str0ng!Pass';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'remote_user'@'localhost';
DROP USER 'remote_user'@'%';
```

## Security Best Practices

- Prefer a specific IP or subnet over `%` wildcard for production users
- Use SSL/TLS for all remote connections
- Rotate passwords regularly
- Limit privileges to only what the remote application needs

## Summary

Creating a MySQL remote access user requires configuring the server's `bind-address`, creating the account with a non-localhost host specification (`%`, an IP, or a subnet), granting the necessary privileges, and opening the firewall port. Always prefer a specific IP restriction over `%` and enable SSL/TLS to protect credentials and data in transit.
