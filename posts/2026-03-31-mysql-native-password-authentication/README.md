# How to Use mysql_native_password Authentication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Authentication, Security, Password, Database Administration

Description: Learn when and how to use the mysql_native_password plugin in MySQL, how it differs from caching_sha2_password, and migration considerations.

---

## What Is mysql_native_password?

`mysql_native_password` is MySQL's legacy authentication plugin, used as the default in MySQL 5.7 and earlier. It stores a SHA1 hash of the user's password. In MySQL 8.0, the default switched to `caching_sha2_password`, which is more secure, but `mysql_native_password` remains available for backward compatibility with older clients and connectors.

## When to Use mysql_native_password

Use `mysql_native_password` when:
- Older client libraries or connectors do not support `caching_sha2_password`
- Third-party tools (some BI or admin tools) require the older auth method
- You are running a mixed 5.7/8.0 environment during migration

## Creating a User with mysql_native_password

```sql
CREATE USER 'legacy_app'@'%'
  IDENTIFIED WITH mysql_native_password
  BY 'AppPass!Legacy2024';
```

## Changing an Existing User to mysql_native_password

```sql
ALTER USER 'alice'@'localhost'
  IDENTIFIED WITH mysql_native_password
  BY 'AliceNewPass!';
```

## Setting the Global Default Authentication Plugin

To make all new accounts use `mysql_native_password` by default (not recommended for new deployments):

In `my.cnf`:

```text
[mysqld]
default_authentication_plugin = mysql_native_password
```

Or dynamically (MySQL 8.0, deprecated in 8.0.27):

```sql
SET GLOBAL default_authentication_plugin = 'mysql_native_password';
```

## Checking Which Plugin a User Uses

```sql
SELECT user, host, plugin
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema');
```

Output:

```text
+---------+-----------+-----------------------------+
| user    | host      | plugin                      |
+---------+-----------+-----------------------------+
| root    | localhost | caching_sha2_password       |
| alice   | localhost | caching_sha2_password       |
| legacy  | %         | mysql_native_password       |
+---------+-----------+-----------------------------+
```

## Security Differences

| Aspect | mysql_native_password | caching_sha2_password |
|--------|-----------------------|----------------------|
| Hash algorithm | SHA1 | SHA256 |
| Brute-force resistance | Lower | Higher |
| Requires SSL for first auth | No | Yes (or RSA key) |
| Supported MySQL versions | 5.0+ | 8.0+ |

## Migrating from mysql_native_password to caching_sha2_password

```sql
ALTER USER 'legacy_app'@'%'
  IDENTIFIED WITH caching_sha2_password
  BY 'AppPass!New2024';
```

Update the client or connector to a version that supports `caching_sha2_password` before making this change. Most modern connectors (MySQL Connector/J 8.0+, mysql2 for Node.js, PyMySQL, etc.) support it.

## Deprecation in MySQL 8.4

MySQL 8.4 removed `mysql_native_password` as an option. Plan migrations to `caching_sha2_password` before upgrading to MySQL 8.4 or later.

```bash
# Check which users still use the legacy plugin before upgrading
mysql -u root -p -e \
  "SELECT user, host FROM mysql.user WHERE plugin='mysql_native_password';"
```

## Summary

`mysql_native_password` provides backward compatibility with older MySQL clients but uses a weaker SHA1-based hash compared to `caching_sha2_password`. Use it only when required by legacy tools, specify it explicitly with `IDENTIFIED WITH mysql_native_password`, and plan a migration to `caching_sha2_password` before upgrading to MySQL 8.4 or later.
