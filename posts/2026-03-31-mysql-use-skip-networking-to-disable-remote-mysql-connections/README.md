# How to Use skip-networking to Disable Remote MySQL Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Networking, Configuration, Database Security

Description: Learn how to use the skip-networking option in MySQL to disable all TCP/IP connections and allow only local socket connections for maximum security.

---

The `skip-networking` option in MySQL disables the TCP/IP stack entirely. When enabled, MySQL stops listening on any network port and accepts connections only through the Unix socket file (on Linux/macOS) or named pipes (on Windows). This is the strongest possible network isolation for a MySQL server.

## When to Use skip-networking

Use `skip-networking` when:
- The application and database server run on the same host
- You want to eliminate any risk of network-based attacks
- You need to comply with security policies that require no open network ports
- You are running a local development or testing environment

Do not use `skip-networking` if:
- Any client needs to connect from a remote host
- You use replication (replicas connect over TCP/IP)
- You have multiple application servers connecting to the database

## Check Current Networking Status

Before making changes, verify whether networking is already disabled:

```sql
SHOW VARIABLES LIKE 'skip_networking';
```

If the value is `OFF`, TCP/IP is currently enabled.

## Enable skip-networking

Edit the MySQL configuration file:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add the following under the `[mysqld]` section:

```text
[mysqld]
skip-networking
```

Restart MySQL to apply the change:

```bash
sudo systemctl restart mysql
```

## Verify the Change

After restarting, confirm that MySQL is no longer listening on port 3306:

```bash
sudo ss -tlnp | grep 3306
```

With `skip-networking` enabled, this command produces no output. You can also verify from inside MySQL:

```sql
SHOW VARIABLES LIKE 'skip_networking';
```

Expected output:

```text
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| skip_networking | ON    |
+-----------------+-------+
```

## Connect Using the Unix Socket

When `skip-networking` is enabled, clients must connect via the Unix socket file rather than a host and port:

```bash
mysql -u root -p --socket=/var/run/mysqld/mysqld.sock
```

Or simply omit the host flag, which causes the MySQL client to default to the socket:

```bash
mysql -u root -p
```

The default socket path can be confirmed:

```sql
SHOW VARIABLES LIKE 'socket';
```

## Connect from an Application

Application connection strings must use the socket path instead of a host/port. Example for a Python application using `mysql-connector-python`:

```python
import mysql.connector

conn = mysql.connector.connect(
    user='appuser',
    password='secret',
    unix_socket='/var/run/mysqld/mysqld.sock',
    database='mydb'
)
```

For PHP with PDO:

```php
$dsn = 'mysql:unix_socket=/var/run/mysqld/mysqld.sock;dbname=mydb';
$pdo = new PDO($dsn, 'appuser', 'secret');
```

## Confirm Remote Connections are Rejected

From a remote machine, attempt a TCP connection. It should fail immediately:

```bash
mysql -u root -p -h <server-ip> -P 3306
```

Expected error:

```text
ERROR 2003 (HY000): Can't connect to MySQL server on '<server-ip>' (111 "Connection refused")
```

## Difference Between skip-networking and bind-address

Both options restrict network access, but they work differently:

```text
skip-networking  - Disables TCP/IP entirely; socket connections only
bind-address     - Still uses TCP/IP but limits which interface is exposed
```

`skip-networking` provides stronger isolation because no port is open at all, while `bind-address = 127.0.0.1` still exposes port 3306 on the loopback interface.

## Summary

The `skip-networking` option in MySQL completely disables TCP/IP networking, forcing all connections to go through the Unix socket. This is the most secure configuration for a single-server setup where the application and database share the same host. Enable it by adding `skip-networking` to the `[mysqld]` section of `mysqld.cnf` and restarting MySQL. Clients then connect using `--socket` or by omitting the host flag entirely.
