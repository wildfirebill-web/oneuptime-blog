# What Is the Error Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error Log, Logging, Diagnostics, Operations

Description: The MySQL error log records server startup and shutdown messages, critical errors, and warnings, making it the first place to look when MySQL has problems.

---

## Overview

The error log is MySQL's primary diagnostic log. It records events related to the server's lifecycle and health: startup and shutdown messages, critical errors that prevent the server from running, recoverable errors and warnings, and information about replica status. Unlike the general query log or slow query log, the error log is always enabled and cannot be turned off. It is the first place to look when troubleshooting MySQL problems.

## Locating the Error Log

```sql
-- Find the error log file path
SHOW VARIABLES LIKE 'log_error';
```

Default locations:
- **Linux (package install)**: `/var/log/mysql/error.log` or `/var/log/mysqld.log`
- **macOS (Homebrew)**: `/usr/local/var/mysql/hostname.err`
- **Windows**: `C:\ProgramData\MySQL\MySQL Server 8.0\Data\hostname.err`

## Configuring the Error Log

In `my.cnf`:

```ini
[mysqld]
log_error = /var/log/mysql/error.log
log_error_verbosity = 2
```

`log_error_verbosity` controls detail level:
- `1`: Errors only
- `2`: Errors and warnings (default)
- `3`: Errors, warnings, and notes

## Typical Error Log Contents

A normal startup sequence:

```
2026-03-31T08:00:00.000000Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.35) starting as process 1234
2026-03-31T08:00:00.500000Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2026-03-31T08:00:01.234567Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2026-03-31T08:00:01.345678Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections.
```

An example warning about a misconfigured setting:

```
2026-03-31T09:15:00.000000Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
```

A crash recovery message:

```
2026-03-31T09:20:00.000000Z 0 [ERROR] [MY-012929] [InnoDB] Corruption in the InnoDB tablespace...
```

## Using the Error Log for Replication Monitoring

The error log records replica thread status changes, making it useful for monitoring replication issues:

```
[ERROR] [MY-010584] [Repl] Replica SQL: Error executing row event: 'Duplicate entry '123'...
[Warning] [MY-010584] [Repl] Replica I/O thread retrying connection after 60 second delay
```

## Error Log in MySQL 8.0 Component Architecture

MySQL 8.0 introduced a component-based error logging architecture. The default log sink is the traditional error log file. You can add JSON format or syslog output:

```sql
-- Install the JSON log sink
INSTALL COMPONENT 'file://component_log_sink_json';

-- Configure which sinks to use
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal; log_sink_json';
```

JSON output makes the log machine-parseable for log aggregation systems.

## Rotating the Error Log

```sql
-- Tell MySQL to close and reopen the error log (after rotating)
FLUSH ERROR LOGS;
```

On Linux, configure `logrotate`:

```ini
/var/log/mysql/error.log {
    daily
    rotate 7
    compress
    postrotate
        mysqladmin -u root -p flush-logs error
    endscript
}
```

## What to Look for When Troubleshooting

| Symptom | Look for in Error Log |
|---|---|
| MySQL won't start | `[ERROR]` messages near startup timestamp |
| InnoDB corruption | `InnoDB: Corruption` messages |
| Replication stopped | `[ERROR] ... Replica SQL` messages |
| Out of memory | `Out of memory` or `Cannot allocate` messages |
| Table crashes | `Table './mydb/tablename' is marked as crashed` |

## Summary

The MySQL error log is always enabled and serves as the central record of server health, startup/shutdown events, critical errors, and warnings. It is the first diagnostic resource for any MySQL problem. Configuring appropriate verbosity, log rotation, and optionally JSON output for log aggregation ensures the error log is both informative and manageable in production environments.
