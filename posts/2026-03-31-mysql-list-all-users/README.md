# How to List All Users in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Database Administration, Security, Information Schema

Description: Learn how to list all MySQL user accounts by querying mysql.user and information_schema, filter by host, and identify system accounts vs application accounts.

---

## The Quick Way - Query mysql.user

The `mysql.user` table stores all user accounts. To list all user-host pairs:

```sql
SELECT user, host FROM mysql.user ORDER BY user, host;
```

Sample output:

```text
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| alice            | localhost |
| bob              | %         |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
```

## Excluding System Accounts

System accounts (`mysql.infoschema`, `mysql.session`, `mysql.sys`) are internal and should usually be excluded from application-level audits:

```sql
SELECT user, host
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema')
ORDER BY user, host;
```

## Viewing More Account Details

```sql
SELECT user, host, plugin, account_locked, password_expired, password_last_changed
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema')
ORDER BY user;
```

This shows the authentication plugin, whether the account is locked, and when the password was last changed - useful for a security audit.

## Using SHOW USERS (Not a Standard Command)

MySQL does not have a `SHOW USERS` command. The query against `mysql.user` is the correct method.

## Listing Users via information_schema

An alternative that does not require SELECT on `mysql.user`:

```sql
SELECT GRANTEE
FROM information_schema.USER_PRIVILEGES
GROUP BY GRANTEE
ORDER BY GRANTEE;
```

This shows accounts as `'user'@'host'` strings and works for users without direct access to the `mysql` system database.

## Counting Total User Accounts

```sql
SELECT COUNT(*) AS total_accounts
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema');
```

## Finding Users by Host Pattern

```sql
-- Users that can connect from anywhere
SELECT user, host FROM mysql.user WHERE host = '%';

-- Users restricted to localhost
SELECT user, host FROM mysql.user WHERE host = 'localhost';

-- Users from a specific subnet
SELECT user, host FROM mysql.user WHERE host LIKE '192.168.%';
```

## Finding Users with No Password

```sql
SELECT user, host
FROM mysql.user
WHERE authentication_string = ''
  AND user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema');
```

These accounts can connect without a password - a significant security risk in production.

## Listing All Accounts from the Shell

```bash
mysql -u root -p -e \
  "SELECT user, host, account_locked FROM mysql.user ORDER BY user, host;"
```

## Finding Duplicate Usernames Across Hosts

The same username can exist with multiple host values - each is a separate account:

```sql
SELECT user, GROUP_CONCAT(host ORDER BY host) AS hosts, COUNT(*) AS count
FROM mysql.user
GROUP BY user
HAVING count > 1
ORDER BY user;
```

## Summary

To list all MySQL users, query `SELECT user, host FROM mysql.user ORDER BY user, host`. Exclude internal system accounts using a `WHERE user NOT IN (...)` clause. For security audits, expand the query to include `plugin`, `account_locked`, `password_expired`, and `password_last_changed` columns. Use `information_schema.USER_PRIVILEGES` as an alternative when you do not have direct access to the `mysql` system database.
