# How to Use skip-networking to Disable Remote MySQL Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Security, Network, Configuration, Administration

Description: Learn how to use the skip-networking option in MySQL to disable all TCP/IP connections and allow only local Unix socket connections for maximum security.

---

## What Is skip-networking?

The `skip-networking` option disables MySQL's TCP/IP listener entirely. With this option enabled, MySQL does not open a network socket and cannot be reached over the network at all. All connections must use the local Unix domain socket (on Linux/macOS) or named pipe (on Windows).

This is the most restrictive network security setting for MySQL and is appropriate for servers where MySQL only needs to be accessed by processes on the same machine.

## Enabling skip-networking

Add to `my.cnf`:

```text
[mysqld]
skip-networking
```

Or with explicit value:

```text
[mysqld]
skip_networking = ON
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## Verifying skip-networking Is Active

After enabling, MySQL should no longer listen on port 3306:

```bash
sudo ss -tlnp | grep mysql
```

With `skip-networking` active, there should be no TCP listener for MySQL. Without it:

```text
LISTEN  0  151  0.0.0.0:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=21))
```

Also verify via MySQL variable:

```sql
SHOW VARIABLES LIKE 'skip_networking';
```

```text
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| skip_networking | ON    |
+-----------------+-------+
```

## Connecting via Unix Socket with skip-networking

When `skip-networking` is active, connect using the socket file:

```bash
mysql -u root -p --socket=/var/run/mysqld/mysqld.sock
```

Or simply:

```bash
mysql -u root -p
```

The MySQL client defaults to the socket when no `--host` is specified (or when `--host=localhost` is used, not `--host=127.0.0.1`).

The socket file location is configured by:

```text
[mysqld]
socket = /var/run/mysqld/mysqld.sock
```

## Using skip-networking with Application Servers

If the application server is on the same machine as MySQL, use the socket in the connection string:

```python
import mysql.connector

conn = mysql.connector.connect(
    unix_socket='/var/run/mysqld/mysqld.sock',
    user='app_user',
    password='SecurePass!',
    database='shop'
)
```

```php
<?php
$pdo = new PDO(
    'mysql:unix_socket=/var/run/mysqld/mysqld.sock;dbname=shop',
    'app_user',
    'SecurePass!'
);
```

## skip-networking vs. bind-address = 127.0.0.1

| Option | TCP Listener | Remote Connections | localhost TCP |
|---|---|---|---|
| `skip-networking` | None | Not possible | Not possible |
| `bind-address = 127.0.0.1` | localhost only | Not possible | Allowed |

Use `skip-networking` when you want zero TCP exposure. Use `bind-address = 127.0.0.1` when some tools or drivers only support TCP (not Unix sockets).

## Enabling Named Pipe on Windows (Alternative)

On Windows, `skip-networking` disables TCP but connections can still use named pipes:

```text
[mysqld]
skip-networking
enable-named-pipe
```

Connect via named pipe:

```text
mysql -u root -p --protocol=PIPE
```

## When to Use skip-networking

- Single-server applications where MySQL and the app are on the same host
- Docker containers where MySQL is an internal service not exposed to the host network
- Development environments where remote access is never needed
- Any situation where network attack surface should be minimized

## Summary

`skip-networking` completely disables MySQL's TCP/IP listener, preventing any network-based connections. Connections are only possible via Unix socket (Linux) or named pipe (Windows). This is the most secure network configuration for MySQL servers that only serve local applications. For cases where TCP localhost access is needed by specific tools, use `bind-address = 127.0.0.1` instead.
