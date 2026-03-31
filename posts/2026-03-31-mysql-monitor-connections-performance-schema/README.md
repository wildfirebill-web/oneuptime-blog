# How to Monitor MySQL Connections with Performance Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Connection, Monitoring, Thread

Description: Use MySQL Performance Schema to monitor active connections, thread states, and per-account connection statistics to diagnose connection-related bottlenecks.

---

## Overview

Connection management is critical in MySQL. Too many idle connections waste memory, while connection bursts can overwhelm the server. Performance Schema provides tables to inspect active threads, connection counts per user, and historical connection patterns.

## Viewing Active Threads

The `threads` table shows every server thread, including background threads and client connections:

```sql
SELECT
  THREAD_ID,
  NAME,
  TYPE,
  PROCESSLIST_USER,
  PROCESSLIST_HOST,
  PROCESSLIST_DB,
  PROCESSLIST_COMMAND,
  PROCESSLIST_TIME,
  PROCESSLIST_STATE
FROM performance_schema.threads
WHERE TYPE = 'FOREGROUND'
ORDER BY PROCESSLIST_TIME DESC;
```

## Monitoring Per-Account Connection Counts

The `accounts` table tracks connections per user/host combination:

```sql
SELECT
  USER,
  HOST,
  CURRENT_CONNECTIONS,
  TOTAL_CONNECTIONS
FROM performance_schema.accounts
WHERE USER IS NOT NULL
ORDER BY CURRENT_CONNECTIONS DESC;
```

## Per-User and Per-Host Summaries

For aggregated views:

```sql
-- By user
SELECT USER, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS
FROM performance_schema.users
ORDER BY CURRENT_CONNECTIONS DESC;

-- By host
SELECT HOST, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS
FROM performance_schema.hosts
ORDER BY CURRENT_CONNECTIONS DESC;
```

## Tracking Connection Events

The `events_waits_current` and `events_waits_history` tables capture connection-related wait events:

```sql
SELECT
  t.PROCESSLIST_USER,
  t.PROCESSLIST_HOST,
  ew.EVENT_NAME,
  ew.TIMER_WAIT / 1e9 AS wait_ms
FROM performance_schema.events_waits_current ew
JOIN performance_schema.threads t USING (THREAD_ID)
WHERE ew.EVENT_NAME LIKE '%connect%'
ORDER BY ew.TIMER_WAIT DESC;
```

## Identifying Long-Running Connections

Find connections that have been idle for a long time:

```sql
SELECT
  PROCESSLIST_ID,
  PROCESSLIST_USER,
  PROCESSLIST_HOST,
  PROCESSLIST_COMMAND,
  PROCESSLIST_TIME,
  PROCESSLIST_STATE
FROM performance_schema.threads
WHERE TYPE = 'FOREGROUND'
  AND PROCESSLIST_COMMAND = 'Sleep'
  AND PROCESSLIST_TIME > 300
ORDER BY PROCESSLIST_TIME DESC;
```

## Using sys Schema Connection Views

```sql
-- Current connections with detailed info
SELECT * FROM sys.session LIMIT 20;

-- Sessions by user
SELECT * FROM sys.user_summary\G
```

## Checking Global Connection Variables

```sql
SHOW STATUS LIKE 'Threads_%';
SHOW STATUS LIKE 'Connection_errors_%';
SHOW VARIABLES LIKE 'max_connections';
```

## Summary

Performance Schema connection monitoring through the `threads`, `accounts`, `users`, and `hosts` tables gives you full visibility into who is connected, for how long, and what they are doing. Combine this with `sys.session` for a comprehensive connection health dashboard that helps prevent connection exhaustion and identify resource-heavy clients.
