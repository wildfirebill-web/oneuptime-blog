# How to Enable the MySQL Event Scheduler

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event Scheduler, Configuration, Automation, Cron

Description: Learn how to enable the MySQL Event Scheduler at runtime and permanently via my.cnf, check its status, and grant the EVENT privilege.

---

The MySQL Event Scheduler is a built-in job scheduler that executes SQL statements at scheduled times, similar to a cron job but managed entirely within MySQL. It is disabled by default on some installations and must be enabled before you can create or run events.

## Check the Current Status

```sql
SHOW VARIABLES LIKE 'event_scheduler';
```

Possible values:
- `ON` - the scheduler is running
- `OFF` - the scheduler is stopped
- `DISABLED` - the scheduler was disabled at startup and cannot be turned on at runtime

## Enable at Runtime

To enable the scheduler for the current server session without restarting MySQL:

```sql
SET GLOBAL event_scheduler = ON;
```

This change takes effect immediately but does not survive a server restart.

Verify:

```sql
SHOW VARIABLES LIKE 'event_scheduler';
-- event_scheduler | ON
```

## Enable Permanently via my.cnf

To make the setting persistent, add it to the MySQL configuration file:

```ini
[mysqld]
event_scheduler = ON
```

The configuration file location varies by OS:

```bash
# Linux - common locations
/etc/mysql/my.cnf
/etc/my.cnf
/etc/mysql/mysql.conf.d/mysqld.cnf

# macOS (Homebrew)
/opt/homebrew/etc/my.cnf

# Windows
C:\ProgramData\MySQL\MySQL Server 8.0\my.ini
```

After editing the file, restart MySQL:

```bash
# Linux systemd
sudo systemctl restart mysql

# macOS Homebrew
brew services restart mysql
```

## Disabling the Scheduler

To stop the scheduler without restarting MySQL:

```sql
SET GLOBAL event_scheduler = OFF;
```

To prevent it from starting at all, set `event_scheduler = DISABLED` in `my.cnf`. Once set to `DISABLED`, it cannot be turned on with `SET GLOBAL`.

## Granting the EVENT Privilege

To allow a user to create and manage events, grant the `EVENT` privilege:

```sql
GRANT EVENT ON mydb.* TO 'scheduler_user'@'localhost';
FLUSH PRIVILEGES;
```

To allow a user to view and manage events across all databases:

```sql
GRANT EVENT ON *.* TO 'dba_user'@'localhost';
```

## Verify the Scheduler is Running

```sql
SELECT * FROM information_schema.PROCESSLIST
WHERE USER = 'event_scheduler';
```

A row with `USER = 'event_scheduler'` confirms the background thread is active.

## Summary

Enable the MySQL Event Scheduler at runtime with `SET GLOBAL event_scheduler = ON` and make it permanent by adding `event_scheduler = ON` to the `[mysqld]` section of `my.cnf`. Grant the `EVENT` privilege to users who need to create scheduled jobs, and verify the scheduler thread is active via `information_schema.PROCESSLIST`.
