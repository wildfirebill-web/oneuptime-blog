# How to Configure MySQL to Listen on a Specific IP Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Bind Address, Network Configuration, Security, My.cnf

Description: Configure MySQL to listen on a specific IP address using the bind-address option in my.cnf to restrict network access and improve security.

---

## Why Limit the Listening Interface

By default, MySQL may bind to all available network interfaces (`0.0.0.0`), which makes it reachable from any network address. Restricting MySQL to a specific IP address reduces the attack surface and prevents unauthorized remote connections.

## Understanding bind-address

The `bind-address` option in `my.cnf` controls which network interface MySQL listens on. Common values include:

- `127.0.0.1` - localhost only (no remote connections)
- `0.0.0.0` - all IPv4 interfaces
- `::` - all IPv4 and IPv6 interfaces
- A specific IP like `192.168.1.50` - only that interface

## Configuring bind-address in my.cnf

Open the MySQL configuration file:

```bash
sudo nano /etc/mysql/my.cnf
# or on some systems:
sudo nano /etc/my.cnf
```

Add or edit the `bind-address` directive under `[mysqld]`:

```text
[mysqld]
bind-address = 127.0.0.1
```

To allow connections from a specific private network IP:

```text
[mysqld]
bind-address = 192.168.1.50
```

Save the file and restart MySQL:

```bash
sudo systemctl restart mysql
```

## Binding to Multiple Addresses (MySQL 8.0+)

MySQL 8.0.13 and later support binding to multiple IP addresses using a comma-separated list:

```text
[mysqld]
bind-address = 127.0.0.1,192.168.1.50
```

This allows local connections and connections from a specific private network interface without exposing MySQL to all interfaces.

## Verifying the Listening Address

After restarting MySQL, verify which address it is listening on:

```bash
sudo ss -tlnp | grep 3306
```

Example output when bound to `127.0.0.1`:

```text
LISTEN  0  151  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=21))
```

Example output when bound to all interfaces:

```text
LISTEN  0  151  0.0.0.0:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=21))
```

## Checking with netstat

Alternatively, use `netstat`:

```bash
netstat -tlnp | grep 3306
```

## Creating User Accounts for Specific Hosts

Restricting `bind-address` controls which interface MySQL listens on, but you also need to grant users the right to connect from specific hosts. These are separate controls.

Create a user that can only connect from a specific IP:

```sql
CREATE USER 'appuser'@'10.0.0.5' IDENTIFIED BY 'StrongPassword!';
GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'10.0.0.5';
FLUSH PRIVILEGES;
```

A user with `'appuser'@'%'` can connect from any host, but only if MySQL is listening on the appropriate interface.

## Using a Firewall Alongside bind-address

For defense in depth, combine `bind-address` with a firewall rule using `ufw` or `iptables`:

```bash
# Allow MySQL only from a specific application server IP
sudo ufw allow from 10.0.0.5 to any port 3306

# Deny all other MySQL connections
sudo ufw deny 3306
```

This ensures that even if `bind-address` is misconfigured, the firewall blocks unwanted traffic.

## Troubleshooting Connection Refused Errors

If clients cannot connect after changing `bind-address`, check:

1. Is MySQL listening on the expected address?

```bash
sudo ss -tlnp | grep 3306
```

2. Does the user account allow connections from the client host?

```sql
SELECT user, host FROM mysql.user WHERE user = 'appuser';
```

3. Is a firewall blocking the port?

```bash
sudo ufw status
```

4. Did MySQL restart successfully after the config change?

```bash
sudo systemctl status mysql
sudo tail -30 /var/log/mysql/error.log
```

## Summary

To restrict MySQL to a specific IP address, set `bind-address` in `my.cnf` under `[mysqld]` to the desired interface IP and restart the service. For localhost-only access use `127.0.0.1`. MySQL 8.0.13+ supports comma-separated values for multiple interfaces. Combine `bind-address` with proper user host restrictions and firewall rules for comprehensive network security.
