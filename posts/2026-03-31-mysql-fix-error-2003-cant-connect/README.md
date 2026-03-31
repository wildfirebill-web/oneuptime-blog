# How to Fix ERROR 2003 Can't Connect to MySQL Server on Host

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Troubleshooting, Database, Networking

Description: Learn how to diagnose and fix MySQL ERROR 2003 Can't connect to MySQL server on host, covering firewall, bind-address, and service issues.

---

## What Is ERROR 2003?

```
ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.1.100' (111)
```

This error means the MySQL client could not establish a TCP connection to the server. The connection was refused at the network level. This is different from authentication errors - the problem is before you even reach MySQL.

## Step 1: Verify MySQL Is Running

On the server, check whether MySQL is actually running:

```bash
sudo systemctl status mysql
# or
sudo systemctl status mysqld
```

If it is not running, start it:

```bash
sudo systemctl start mysql
```

Check for startup errors in the log:

```bash
sudo journalctl -u mysql -n 50
sudo tail -n 50 /var/log/mysql/error.log
```

## Step 2: Check That MySQL Is Listening on the Right Port

```bash
sudo ss -tlnp | grep 3306
# or
sudo netstat -tlnp | grep 3306
```

If MySQL is not listening on port 3306, or is only listening on `127.0.0.1`, you need to fix the `bind-address`.

## Step 3: Fix the bind-address Setting

By default, MySQL binds to `127.0.0.1` (localhost only), which blocks remote connections.

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf` or `/etc/my.cnf`:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find and change:

```ini
bind-address = 127.0.0.1
```

To either:

```ini
# Listen on all interfaces
bind-address = 0.0.0.0

# Or listen on a specific interface IP
bind-address = 192.168.1.100
```

Restart MySQL after the change:

```bash
sudo systemctl restart mysql
```

## Step 4: Check the Firewall

If MySQL is listening but connections are still refused, a firewall may be blocking port 3306.

On Ubuntu with UFW:

```bash
sudo ufw allow 3306/tcp
sudo ufw status
```

On CentOS/RHEL with firewalld:

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

## Step 5: Test Connectivity from the Client

From the client machine, test TCP connectivity before attempting a MySQL connection:

```bash
nc -zv 192.168.1.100 3306
# or
telnet 192.168.1.100 3306
```

If this fails, the issue is at the network or firewall level, not MySQL.

## Step 6: Grant Remote Access to the MySQL User

Even after fixing the network, the user must have a host-level grant that allows remote connections:

```sql
-- Check existing grants
SHOW GRANTS FOR 'myuser'@'%';

-- Grant access from any host
CREATE USER 'myuser'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'%';
FLUSH PRIVILEGES;
```

A user created as `'myuser'@'localhost'` cannot connect remotely. You need `'myuser'@'%'` or a specific host.

## Common Error Codes in the Message

| Code | Meaning |
|---|---|
| 111 | Connection refused (port closed or MySQL not running) |
| 110 | Connection timed out (firewall blocking, no route) |
| 61 | Connection refused (macOS equivalent of 111) |

## Summary

ERROR 2003 is a TCP-level connection failure. Work through this checklist: verify MySQL is running, check that it is listening on the correct address and port, open the firewall on port 3306, test connectivity with `nc` or `telnet`, and ensure the MySQL user has a remote host grant. Most cases are resolved by fixing `bind-address` or the firewall rule.
