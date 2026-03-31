# How to Troubleshoot MySQL Startup Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Startup, InnoDB, Configuration, Troubleshooting

Description: Learn how to diagnose and fix MySQL startup failures by reading error logs, checking permissions, resolving port conflicts, and using InnoDB recovery mode.

---

## Where to Look First

When MySQL fails to start, the error log contains the most detail. Find it with:

```bash
mysqld --verbose --help 2>/dev/null | grep "^log-error"
# Or check the config
grep -i "log.error" /etc/mysql/mysql.conf.d/mysqld.cnf
```

Tail the log immediately after a failed start attempt:

```bash
journalctl -u mysql --since "5 minutes ago"
# Or
tail -100 /var/log/mysql/error.log
```

## Common Cause 1: Port Already in Use

```text
[ERROR] Can't start server: Bind on TCP/IP port. Got error: 98: Address already in use
```

Check what is using port 3306:

```bash
ss -tlnp | grep 3306
lsof -i :3306
```

If a previous MySQL instance is still running:

```bash
systemctl stop mysql
killall mysqld
systemctl start mysql
```

## Common Cause 2: Data Directory Permission Issues

```text
[ERROR] Fatal error: Can't open and lock privilege tables: Table 'mysql.user' doesn't exist
[ERROR] --initialize specified but the data directory has files in it.
```

Check ownership of the data directory:

```bash
ls -la /var/lib/mysql
# All files should be owned by mysql:mysql
chown -R mysql:mysql /var/lib/mysql
chmod 750 /var/lib/mysql
```

## Common Cause 3: InnoDB Recovery Required

```text
InnoDB: Database was not shut down normally!
InnoDB: Starting crash recovery.
```

If crash recovery fails, enable forced recovery:

```bash
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
innodb_force_recovery = 1
```

Start MySQL, dump data, then remove the option and reinitialize cleanly.

## Common Cause 4: Configuration Errors

Typos in `my.cnf` prevent startup with messages like:

```text
[ERROR] unknown variable 'innodb_buffer_pool_sizee=2G'
```

Validate the configuration without starting:

```bash
mysqld --validate-config
```

Or check with:

```bash
my_print_defaults mysqld
```

## Common Cause 5: Disk Full

```text
[ERROR] InnoDB: Operating system error number 28 in a file operation.
InnoDB: Error number 28 means 'No space left on device'.
```

Check disk space:

```bash
df -h /var/lib/mysql
```

Free space by purging binary logs, removing old temporary files, or expanding the volume, then restart MySQL.

## Common Cause 6: Socket File Conflict

```text
[ERROR] Can't create/write to file '/var/run/mysqld/mysqld.sock'
```

Remove a stale socket file and check directory permissions:

```bash
rm -f /var/run/mysqld/mysqld.sock
mkdir -p /var/run/mysqld
chown mysql:mysql /var/run/mysqld
systemctl start mysql
```

## Verifying the Fix

After resolving the issue, confirm MySQL is running and accepting connections:

```bash
systemctl status mysql
mysqladmin -u root -p status
```

Check the error log for any remaining warnings after startup:

```bash
grep -i "warning\|error" /var/log/mysql/error.log | tail -20
```

## Summary

MySQL startup failures are diagnosed primarily through the error log. The most common causes are port conflicts, incorrect file permissions, InnoDB recovery requirements, configuration syntax errors, a full disk, and stale socket files. Use `mysqld --validate-config` to catch configuration errors before attempting a restart, and use `innodb_force_recovery` incrementally when the server cannot start due to InnoDB corruption.
