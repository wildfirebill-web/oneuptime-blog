# What Is the General Query Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, General Query Log, Logging, Debugging, Audit

Description: The MySQL general query log records every SQL statement received from clients, including connection events, making it useful for debugging and auditing.

---

## Overview

The general query log is a MySQL log that records all SQL statements sent to the server by clients, in the order they arrive. Unlike the slow query log (which only captures slow queries) or the binary log (which captures committed changes), the general query log captures everything: `SELECT` queries, `SET` commands, connection and disconnection events, and failed queries. Because it records statements as they arrive rather than when they commit, it is more verbose than the binary log.

The general query log is disabled by default because it has a significant performance impact when all queries on a busy server are logged to disk.

## Enabling the General Query Log

```sql
-- Enable at runtime (does not persist after restart)
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/general.log';

-- Check current status
SHOW VARIABLES LIKE 'general_log%';
```

To enable persistently in `my.cnf`:

```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/general.log
```

## Log Output Destination

Control where the general log is written:

```sql
-- Log to a file (default)
SET GLOBAL log_output = 'FILE';

-- Log to the mysql.general_log table
SET GLOBAL log_output = 'TABLE';

-- Log to both
SET GLOBAL log_output = 'FILE,TABLE';
```

When logging to the `mysql.general_log` table:

```sql
SELECT event_time, user_host, command_type, argument
FROM mysql.general_log
ORDER BY event_time DESC
LIMIT 20;
```

## What the General Query Log Records

A typical general log entry looks like:

```
2026-03-31T10:15:30.123456Z    42 Connect   app_user@localhost on mydb
2026-03-31T10:15:30.124567Z    42 Query     SELECT id, name FROM products WHERE id = 7
2026-03-31T10:15:30.125678Z    42 Query     UPDATE orders SET status = 'shipped' WHERE id = 101
2026-03-31T10:15:30.126789Z    42 Quit
```

Each line includes: timestamp, thread ID, command type, and the SQL text.

Command types include:
- `Connect` / `Quit`: connection lifecycle events
- `Query`: SQL statements
- `Prepare` / `Execute` / `Close stmt`: prepared statement lifecycle
- `Init DB`: `USE database` commands

## Filtering by User or Database

For targeted logging (MySQL 8.0+), you can enable logging for specific users:

```sql
-- Disable logging for a specific user
SET GLOBAL log_output = 'FILE';
ALTER USER 'monitoring_user'@'localhost' ATTRIBUTE '{"general_log": false}';
```

## When to Use the General Query Log

**Use it for**:
- Short-term debugging: "What queries is this application actually sending?"
- Auditing during security investigations
- Identifying unexpected queries from ORM or legacy code

**Avoid it for**:
- Production servers under heavy load (significant performance impact)
- Long-term auditing (use a dedicated audit plugin instead)

## Performance Impact

The general query log writes synchronously for every statement. On busy servers it can measurably reduce throughput. Enable it only temporarily for debugging windows and disable it when done:

```sql
-- Enable temporarily
SET GLOBAL general_log = ON;

-- ... perform debugging ...

-- Disable when done
SET GLOBAL general_log = OFF;
```

## Summary

The MySQL general query log captures every SQL statement sent to the server, providing complete visibility into client activity including connections, queries, and prepared statements. It is an invaluable debugging and short-term auditing tool but too expensive to run continuously on production servers. Use it in targeted windows when investigating specific issues, and prefer the slow query log or a dedicated audit plugin for ongoing monitoring.
