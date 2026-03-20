# How to Grant MySQL User Access from a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IPv4, GRANT, User Access, Security, Database

Description: Create MySQL users with access restricted to specific IPv4 addresses and subnets using GRANT statements, revoke access, and manage host-based user accounts.

## Introduction

MySQL user accounts include both a username and a host: `'user'@'host'`. The host component defines which IP addresses or hostnames can connect as that user. This provides fine-grained access control per client IP.

## Creating Users with Host Restrictions

```bash
sudo mysql -u root -p

-- Single specific IP
CREATE USER 'appuser'@'10.0.0.5' IDENTIFIED BY 'strongpassword';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'appuser'@'10.0.0.5';

-- Subnet using wildcard (last octet)
CREATE USER 'webuser'@'192.168.1.%' IDENTIFIED BY 'webpass';
GRANT SELECT ON webapp.* TO 'webuser'@'192.168.1.%';

-- Monitoring user (read-only from monitoring server)
CREATE USER 'monitor'@'203.0.113.30' IDENTIFIED BY 'monpass';
GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO 'monitor'@'203.0.113.30';

-- Apply changes
FLUSH PRIVILEGES;
```

## MySQL Host Wildcard Patterns

```sql
-- Single IP
'user'@'10.0.0.5'         -- Exact IP match

-- Last octet wildcard
'user'@'10.0.0.%'         -- 10.0.0.0-10.0.0.255

-- Two-octet wildcard
'user'@'10.0.%'           -- Any 10.0.x.x address

-- Any host (insecure - avoid for production)
'user'@'%'                -- Any IP anywhere

-- Localhost only
'user'@'localhost'        -- Local socket connection
'user'@'127.0.0.1'        -- TCP loopback
```

## Viewing and Managing User Hosts

```sql
-- View all user accounts and their allowed hosts
SELECT User, Host, authentication_string FROM mysql.user ORDER BY User, Host;

-- Check grants for a specific user@host
SHOW GRANTS FOR 'appuser'@'10.0.0.5';

-- Grant additional privileges to existing user
GRANT CREATE TEMPORARY TABLES ON appdb.* TO 'appuser'@'10.0.0.5';

-- Change a user's host (recreate with different host)
RENAME USER 'appuser'@'10.0.0.5' TO 'appuser'@'10.0.0.6';

-- Revoke specific privilege
REVOKE DELETE ON appdb.* FROM 'appuser'@'10.0.0.5';

-- Remove user entirely
DROP USER 'appuser'@'10.0.0.5';

FLUSH PRIVILEGES;
```

## Adding Access from Multiple IPs

```bash
sudo mysql -u root -p

-- Same user, different hosts (separate GRANT statements)
CREATE USER 'appuser'@'10.0.0.5' IDENTIFIED BY 'password';
CREATE USER 'appuser'@'10.0.0.6' IDENTIFIED BY 'password';
CREATE USER 'appuser'@'10.0.0.7' IDENTIFIED BY 'password';

GRANT ALL ON appdb.* TO 'appuser'@'10.0.0.5';
GRANT ALL ON appdb.* TO 'appuser'@'10.0.0.6';
GRANT ALL ON appdb.* TO 'appuser'@'10.0.0.7';

-- Or use subnet wildcard (simpler for many IPs in same subnet)
CREATE USER 'appuser'@'10.0.0.%' IDENTIFIED BY 'password';
GRANT ALL ON appdb.* TO 'appuser'@'10.0.0.%';
```

## Testing Access

```bash
# Test from allowed IP (10.0.0.5)

mysql -h 203.0.113.10 -u appuser -p appdb
# Expected: connection successful

# Test from blocked IP (10.0.0.99 not in grants)
mysql -h 203.0.113.10 -u appuser -p appdb
# Expected: ERROR 1130 (HY000): Host '10.0.0.99' is not allowed to connect

# Diagnose: check if host matches any grant
sudo mysql -e "SELECT User,Host FROM mysql.user WHERE User='appuser';"
```

## Conclusion

MySQL host-based access control ties user accounts to specific IP addresses using `'user'@'ip_or_pattern'`. Create separate user entries for each allowed host or use `%` wildcards for subnet ranges. Use `SHOW GRANTS FOR 'user'@'host'` to audit access, and `DROP USER` to remove access. Always prefer specific IPs over broad wildcards for production database accounts.
