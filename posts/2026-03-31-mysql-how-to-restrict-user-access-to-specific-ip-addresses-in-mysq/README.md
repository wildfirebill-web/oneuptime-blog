# How to Restrict User Access to Specific IP Addresses in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Access Control, IP Restriction, Administration

Description: Learn how to create MySQL user accounts that are restricted to specific IP addresses or subnets, preventing unauthorized access from unknown hosts.

---

## How MySQL Host Restrictions Work

In MySQL, a user account is identified by both the username and the host from which they connect. The host portion of the account name (`'user'@'host'`) can be an IP address, hostname, or wildcard. This means the same username can have different privileges depending on where the connection originates.

## Creating a User Restricted to a Single IP

```sql
CREATE USER 'app_user'@'192.168.1.100' IDENTIFIED BY 'SecurePassword!';
GRANT SELECT, INSERT, UPDATE ON shop.* TO 'app_user'@'192.168.1.100';
```

This user can only connect from `192.168.1.100`. A connection from any other IP with the same credentials will fail with:

```text
ERROR 1045 (28000): Access denied for user 'app_user'@'192.168.1.101'
```

## Restricting to a Subnet with Wildcards

MySQL supports `%` as a wildcard in the host portion:

```sql
-- Allow connections from any host in 10.0.1.x
CREATE USER 'app_user'@'10.0.1.%' IDENTIFIED BY 'SecurePassword!';
GRANT SELECT, INSERT ON shop.* TO 'app_user'@'10.0.1.%';
```

## Restricting to Multiple Specific IPs

MySQL does not support comma-separated IPs in a single account. Create separate accounts for each IP:

```sql
CREATE USER 'app_user'@'10.0.1.10' IDENTIFIED BY 'SecurePassword!';
CREATE USER 'app_user'@'10.0.1.11' IDENTIFIED BY 'SecurePassword!';
CREATE USER 'app_user'@'10.0.1.12' IDENTIFIED BY 'SecurePassword!';

GRANT SELECT, INSERT ON shop.* TO 'app_user'@'10.0.1.10';
GRANT SELECT, INSERT ON shop.* TO 'app_user'@'10.0.1.11';
GRANT SELECT, INSERT ON shop.* TO 'app_user'@'10.0.1.12';
```

## Combining Hostname and IP Restrictions

You can use hostnames instead of IPs. MySQL resolves them at connection time:

```sql
CREATE USER 'app_user'@'app-server.internal' IDENTIFIED BY 'SecurePassword!';
GRANT ALL ON app_db.* TO 'app_user'@'app-server.internal';
```

Note: hostname-based restrictions require DNS resolution and can be slower. Prefer IP-based restrictions for performance and reliability.

## Checking Existing User Host Restrictions

```sql
SELECT user, host, authentication_string
FROM mysql.user
WHERE user = 'app_user';
```

```text
+----------+-------------+----------------------------------+
| user     | host        | authentication_string            |
+----------+-------------+----------------------------------+
| app_user | 10.0.1.10   | $A$005$...                       |
| app_user | 10.0.1.11   | $A$005$...                       |
| app_user | localhost   | $A$005$...                       |
+----------+-------------+----------------------------------+
```

## Renaming a User to Change the Host

If you need to update the allowed IP:

```sql
RENAME USER 'app_user'@'10.0.1.10' TO 'app_user'@'10.0.2.10';
```

## Dropping a Host-Specific Account

```sql
DROP USER 'app_user'@'10.0.1.10';
```

This only removes the account for that specific IP. Other accounts with the same username but different hosts remain unaffected.

## Combining IP Restrictions with SSL Requirements

For additional security, require SSL for IP-restricted accounts:

```sql
ALTER USER 'app_user'@'10.0.1.10' REQUIRE SSL;
```

Or require a specific X.509 subject:

```sql
ALTER USER 'app_user'@'10.0.1.10'
  REQUIRE SUBJECT '/C=US/O=Acme/CN=app-server';
```

## Summary

MySQL enforces IP-based access control through the host component of user accounts (`'user'@'host'`). Create separate accounts per IP address for strict control, or use subnet wildcards (`10.0.1.%`) for flexible range restrictions. Combine IP restrictions with SSL requirements for defense-in-depth. Use `SELECT user, host FROM mysql.user` to audit existing host restrictions.
