# How to Disable MySQL skip-networking to Enable IPv4 TCP Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IPv4, TCP, Configuration, Networking, Database, skip-networking

Description: Learn how to disable the MySQL skip-networking option to enable TCP connections over IPv4 and configure the bind address for secure remote access.

---

MySQL's `skip-networking` option disables all TCP/IP connections, leaving only Unix socket connections. While this is a security hardening measure for single-server setups, it prevents any remote IPv4 client, replication replica, or application server from connecting over the network.

## Understanding skip-networking

| Configuration | Effect |
|--------------|--------|
| `skip-networking` | Disable all TCP/IP; Unix socket only |
| `bind-address = 127.0.0.1` | Listen on IPv4 loopback only |
| `bind-address = 0.0.0.0` | Listen on all IPv4 interfaces |
| `bind-address = 192.168.1.10` | Listen on one specific IPv4 address |

## Step 1: Check Current Configuration

```bash
# Check if skip-networking is set
grep -r "skip-networking" /etc/mysql/

# Check the current bind-address
mysql -u root -p -e "SHOW VARIABLES LIKE 'skip_networking';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'bind_address';"
```

## Step 2: Remove skip-networking

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf (Ubuntu/Debian)
# or /etc/my.cnf (RHEL/CentOS)

[mysqld]
# Comment out or remove skip-networking
# skip-networking    ← REMOVE OR COMMENT THIS LINE

# Set bind-address to the desired IPv4 address
# 127.0.0.1 = localhost only (safest)
# 0.0.0.0 = all IPv4 interfaces (required for remote access)
# 192.168.1.10 = specific interface
bind-address = 0.0.0.0

# Optional: also disable IPv6 by setting only IPv4 bind
mysqlx-bind-address = 0.0.0.0
```

## Step 3: Restart MySQL

```bash
systemctl restart mysql

# Verify MySQL is listening on TCP
ss -tlnp | grep :3306
# Should show: 0.0.0.0:3306 (IPv4) or :::3306 (both)
```

## Step 4: Create a Remote-Access User

MySQL requires an explicit user grant with the remote IP.

```sql
-- Allow the app user to connect from a specific application server IPv4
CREATE USER 'appuser'@'192.168.1.20' IDENTIFIED BY 'SecurePass123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'appuser'@'192.168.1.20';

-- Or allow from any host on the internal subnet (less restrictive)
CREATE USER 'appuser'@'192.168.1.%' IDENTIFIED BY 'SecurePass123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'appuser'@'192.168.1.%';

FLUSH PRIVILEGES;
```

## Step 5: Open the Firewall

```bash
# Allow MySQL connections from the application subnet only
ufw allow from 192.168.1.0/24 to any port 3306
# or
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -j DROP
```

## Testing Remote Connectivity

```bash
# Test from the application server (192.168.1.20)
mysql -h 192.168.1.10 -u appuser -p mydb

# Or with TCP forced (bypasses Unix socket)
mysql -h 127.0.0.1 -P 3306 -u root -p
```

## Key Takeaways

- Comment out `skip-networking` in `mysqld.cnf` to enable TCP connections.
- Set `bind-address = 0.0.0.0` to listen on all IPv4 interfaces, or a specific IP for stricter control.
- Always create user accounts that include the remote host IP in the grant (e.g., `'user'@'192.168.1.20'`).
- Combine `bind-address` with firewall rules to limit which IPv4 networks can reach the MySQL port.
