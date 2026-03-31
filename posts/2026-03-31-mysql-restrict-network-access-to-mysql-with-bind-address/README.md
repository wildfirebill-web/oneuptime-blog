# How to Restrict Network Access to MySQL with bind-address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Networking, bind-address, Database Security, Configuration

Description: Learn how to use the bind-address option in MySQL to restrict which network interfaces the server listens on, limiting exposure to untrusted networks.

---

By default, MySQL may listen on all available network interfaces, making it reachable from any IP address that can reach the host. The `bind-address` configuration option controls which interface MySQL binds to, providing a first line of defense for network-level access control.

## Understanding bind-address

The `bind-address` option tells MySQL which IP address to listen on for incoming connections. Setting it to a specific IP means MySQL will only accept connections arriving on that interface.

Common values:

```text
127.0.0.1   - Listen on localhost only (IPv4)
::1         - Listen on localhost only (IPv6)
0.0.0.0     - Listen on all IPv4 interfaces (default behavior)
::          - Listen on all IPv4 and IPv6 interfaces
192.168.1.5 - Listen on a specific private network interface only
```

## Check the Current Binding

To see what address MySQL is currently listening on, run this from the shell:

```bash
sudo ss -tlnp | grep mysql
```

Or from inside MySQL:

```sql
SHOW VARIABLES LIKE 'bind_address';
```

## Restrict MySQL to Localhost Only

The most restrictive setting prevents any external connections. This is appropriate when the application and the database run on the same server.

Edit the MySQL configuration file:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Set `bind-address` under the `[mysqld]` section:

```text
[mysqld]
bind-address = 127.0.0.1
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

After this change, any attempt to connect from a remote host will be refused at the network layer before MySQL even processes the login.

## Bind to a Specific Private Interface

When the application runs on a separate server within the same private network, bind MySQL to the private interface only:

```text
[mysqld]
bind-address = 192.168.1.10
```

This allows connections from hosts that can reach `192.168.1.10` while blocking connections on public interfaces.

## Allow Multiple Interfaces (MySQL 8.0.13+)

Starting with MySQL 8.0.13, `bind-address` accepts a comma-separated list of addresses:

```text
[mysqld]
bind-address = 127.0.0.1,192.168.1.10
```

This makes MySQL reachable only on localhost and a specific private IP, not on any public interface.

## Combine bind-address with Firewall Rules

`bind-address` is a server-level control, but it should be layered with firewall rules for defense in depth. Using `ufw` on Ubuntu:

```bash
# Allow MySQL from a specific application server IP only
sudo ufw allow from 192.168.1.20 to any port 3306
# Deny all other MySQL traffic
sudo ufw deny 3306
sudo ufw reload
```

Using `iptables`:

```bash
sudo iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.20 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
```

## Verify After Applying the Change

Confirm MySQL is no longer listening on all interfaces:

```bash
sudo ss -tlnp | grep 3306
```

Expected output with `bind-address = 127.0.0.1`:

```text
LISTEN 0  70  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=24))
```

The address `127.0.0.1` confirms MySQL is bound to localhost only, not `0.0.0.0`.

## Test Remote Connection is Blocked

From a remote machine, attempt a connection. It should fail:

```bash
mysql -u root -p -h <server-public-ip>
```

Expected error:

```text
ERROR 2003 (HY000): Can't connect to MySQL server on '<ip>' (111 "Connection refused")
```

## Summary

The `bind-address` option in MySQL's `mysqld.cnf` controls which network interface MySQL listens on. Setting it to `127.0.0.1` restricts access to localhost only, while a private IP limits access to a specific internal network. For full protection, combine `bind-address` with OS-level firewall rules and MySQL user host restrictions to create multiple layers of network access control.
