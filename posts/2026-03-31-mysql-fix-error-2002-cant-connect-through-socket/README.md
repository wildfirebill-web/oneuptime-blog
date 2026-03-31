# How to Fix ERROR 2002 Can't Connect Through Socket in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection, Socket, Error, Troubleshooting

Description: Fix MySQL ERROR 2002 caused by a missing or wrong socket file path, MySQL not running, or permission issues on the Unix socket file.

---

MySQL ERROR 2002 appears when connecting via a Unix domain socket and the connection cannot be made. The full message reads: `ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)`. The number in parentheses is a system error code: 2 means the file does not exist.

## Check if MySQL Is Running

The most common cause is that MySQL is not running:

```bash
# Check service status
sudo systemctl status mysql
# Or on some distributions
sudo systemctl status mysqld

# Start if stopped
sudo systemctl start mysql
```

If MySQL fails to start, check the error log:

```bash
sudo tail -100 /var/log/mysql/error.log
# Or on RHEL/CentOS
sudo tail -100 /var/log/mysqld.log
```

## Find the Correct Socket Path

The socket file path differs by distribution and installation method:

```bash
# Find the socket location configured in my.cnf
mysql --help | grep "Default options" -A 1
grep -r "socket" /etc/mysql/ 2>/dev/null
grep -r "socket" /etc/my.cnf 2>/dev/null

# Search for the socket file on disk
sudo find /var /tmp -name "mysql.sock" 2>/dev/null
sudo find /var /tmp -name "mysqld.sock" 2>/dev/null
```

Common locations:
- `/var/lib/mysql/mysql.sock`
- `/tmp/mysql.sock`
- `/var/run/mysqld/mysqld.sock`

## Connect Using the Correct Socket

Once you know the socket path, specify it explicitly:

```bash
mysql -u root -p --socket=/var/run/mysqld/mysqld.sock
```

Or add it to your client configuration:

```text
[client]
socket = /var/run/mysqld/mysqld.sock
```

## Fix: Update my.cnf to Match

Ensure both the server and client sections use the same socket path:

```text
[mysqld]
socket = /var/lib/mysql/mysql.sock

[client]
socket = /var/lib/mysql/mysql.sock
```

After editing, restart MySQL:

```bash
sudo systemctl restart mysql
```

## Fix: Permission Issues

The socket file must be readable by the connecting user:

```bash
# Check socket file permissions
ls -la /var/lib/mysql/mysql.sock

# The mysql user should own the socket directory
ls -la /var/lib/mysql/

# Fix permissions on the data directory
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod 755 /var/lib/mysql
```

## Fix: Stale Socket File

If MySQL crashed, a stale socket file may remain:

```bash
# Remove the stale socket file (only if MySQL is stopped)
sudo rm /var/lib/mysql/mysql.sock

# Restart MySQL - it will recreate the socket
sudo systemctl start mysql
```

## Use TCP Instead of Socket

As a workaround, connect via TCP on localhost:

```bash
mysql -u root -p -h 127.0.0.1 -P 3306
```

Note: using `127.0.0.1` forces TCP while `localhost` uses the socket.

## Summary

ERROR 2002 almost always means MySQL is not running or the socket file path does not match between client and server. Check the service status first, then verify the socket path in `my.cnf` matches on both `[mysqld]` and `[client]` sections. Remove stale socket files after crashes and always restart the service after configuration changes.
