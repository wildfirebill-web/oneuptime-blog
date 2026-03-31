# How to Configure MySQL to Listen on a Specific IP Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Configuration, Networking, Bind Address, Database Administration

Description: Configure MySQL to listen on a specific IP address by setting the bind-address option in my.cnf, controlling which network interfaces accept connections.

---

## Why Configure MySQL to Listen on a Specific IP

By default, MySQL listens on all available network interfaces (`0.0.0.0` for IPv4 or `::` for IPv6). This means any host that can reach any IP on your server can attempt to connect to MySQL, which increases the attack surface.

Binding MySQL to a specific IP address restricts which interfaces accept connections. This is useful when:

- You want MySQL to only accept local application connections (bind to `127.0.0.1`)
- Your server has multiple network interfaces (public and private) and MySQL should only be accessible via the private network
- You are segmenting database traffic to a dedicated network interface

## Locating the MySQL Configuration File

The main MySQL configuration file is usually `/etc/mysql/mysql.conf.d/mysqld.cnf` on Ubuntu/Debian or `/etc/my.cnf` on RHEL/CentOS/Amazon Linux.

You can confirm which file MySQL is using:

```bash
mysql --help | grep "Default options" -A 1
```

Example output:

```text
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

## Setting bind-address in my.cnf

Open the MySQL configuration file with a text editor:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find the `bind-address` directive under the `[mysqld]` section and change it to the desired IP:

```text
[mysqld]
bind-address = 127.0.0.1
```

Common values:

| Value | Effect |
|---|---|
| `127.0.0.1` | Only accept local connections (loopback) |
| `10.0.0.5` | Only accept connections to this specific private IP |
| `0.0.0.0` | Accept connections on all IPv4 interfaces |
| `::` | Accept connections on all IPv4 and IPv6 interfaces |

## Binding to a Private IP Address

If your server has a private IP (for example, `10.0.1.100`) and you want application servers on the same private network to connect, but not external traffic:

```text
[mysqld]
bind-address = 10.0.1.100
```

## Binding to Multiple Addresses

MySQL 8.0.13 and later supports binding to multiple addresses using a comma-separated list:

```text
[mysqld]
bind-address = 127.0.0.1,10.0.1.100
```

In older MySQL versions, you would need to use `0.0.0.0` and rely on firewall rules (such as `iptables` or `firewalld`) to restrict which source IPs can reach port 3306.

## Restarting MySQL After the Change

After editing the configuration file, restart MySQL for the change to take effect:

```bash
# On systemd-based systems (Ubuntu, Debian, CentOS 7+, RHEL)
sudo systemctl restart mysqld

# Or on some systems
sudo systemctl restart mysql
```

## Verifying the Binding

After restarting, confirm MySQL is listening on the expected IP and port:

```bash
sudo ss -tlnp | grep 3306
```

Example output showing MySQL bound to `127.0.0.1` only:

```text
LISTEN  0  151  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=23))
```

If `bind-address = 0.0.0.0`, you would see:

```text
LISTEN  0  151  0.0.0.0:3306  0.0.0.0:*  users:(("mysqld",pid=1234,fd=23))
```

## Firewall Rules as a Complement

Even after binding to a specific IP, adding firewall rules provides defense in depth. Using `ufw` on Ubuntu:

```bash
# Allow MySQL connections only from a specific application server IP
sudo ufw allow from 10.0.1.50 to any port 3306

# Deny all other MySQL connections
sudo ufw deny 3306
```

Using `firewall-cmd` on RHEL/CentOS:

```bash
# Allow MySQL from a specific source IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.1.50" port protocol="tcp" port="3306" accept'

sudo firewall-cmd --reload
```

## Testing Connectivity

From the application server, test that MySQL is reachable on the intended IP:

```bash
mysql -h 10.0.1.100 -u appuser -p
```

From any host that should NOT have access, confirm the connection is refused:

```bash
mysql -h 10.0.1.100 -u appuser -p
# Should result in: ERROR 2003 (HY000): Can't connect to MySQL server
```

## Troubleshooting

If MySQL fails to start after changing `bind-address`, check the MySQL error log:

```bash
sudo journalctl -u mysqld -n 50
```

Common issues:
- The specified IP does not exist on any network interface
- Another process is already using port 3306 on that IP
- Firewall rules blocking the port before MySQL can bind

## Summary

Configuring MySQL to listen on a specific IP address is done by setting the `bind-address` option in the `[mysqld]` section of `my.cnf`. Binding to `127.0.0.1` restricts MySQL to local connections only, while binding to a private IP allows connections from internal network hosts. Always pair bind-address configuration with firewall rules for a layered security approach, and restart MySQL after making changes.
