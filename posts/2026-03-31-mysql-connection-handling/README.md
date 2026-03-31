# How MySQL Connection Handling Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection, Performance, Thread

Description: Learn how MySQL handles client connections internally, from TCP handshake to thread allocation, and how to tune settings for high-concurrency workloads.

---

## Overview

Every time a client connects to MySQL, a chain of events unfolds beneath the surface. Understanding this chain helps you tune MySQL for high concurrency, diagnose connection errors, and avoid common pitfalls like connection exhaustion.

## The Connection Lifecycle

When a client initiates a connection, MySQL goes through several steps:

1. The client opens a TCP socket to port 3306 (or a Unix socket for local connections).
2. MySQL's main listener thread accepts the connection and performs the initial handshake.
3. The server sends a handshake packet including server version, connection ID, and supported authentication plugins.
4. The client responds with credentials; MySQL authenticates using the configured plugin (default: `caching_sha2_password` in MySQL 8.x).
5. A dedicated thread is assigned to service the connection.

```text
Client --> TCP SYN --> MySQL Listener
MySQL  --> Handshake Packet --> Client
Client --> Auth Response --> MySQL
MySQL  --> OK / Error Packet --> Client
MySQL  --> Assign Thread --> Connection Established
```

## Thread-Per-Connection Model

By default, MySQL uses a one-thread-per-connection model. Each connection gets its own OS thread for the duration of the session. This means:

- Context switching overhead grows with connection count.
- Each thread consumes memory (stack size is around 256 KB by default).
- Under high concurrency, the thread count itself becomes a bottleneck.

```sql
-- Check current connection stats
SHOW STATUS LIKE 'Threads_%';
-- Threads_connected: active connections
-- Threads_running: threads currently executing queries
-- Threads_cached: threads sitting idle in the thread cache
```

## Thread Cache

To avoid the cost of spawning new OS threads on every connection, MySQL maintains a thread cache controlled by `thread_cache_size`. When a connection closes, its thread returns to the cache instead of being destroyed. The next incoming connection can reuse a cached thread.

```sql
-- Check cache hit rate
SHOW STATUS LIKE 'Connections';
SHOW STATUS LIKE 'Threads_created';
-- If Threads_created / Connections is high, increase thread_cache_size
```

```sql
-- Increase thread cache
SET GLOBAL thread_cache_size = 100;
```

## Key Configuration Variables

```sql
-- Maximum simultaneous connections
SHOW VARIABLES LIKE 'max_connections';

-- Per-user connection limit
SELECT user, max_connections FROM mysql.user;

-- Connection timeout settings
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';
```

A common configuration in `my.cnf`:

```text
[mysqld]
max_connections        = 500
thread_cache_size      = 50
wait_timeout           = 300
interactive_timeout    = 300
connect_timeout        = 10
```

## Connection Errors and Back Log

MySQL maintains a back log queue for connections that arrive while the server is busy accepting others. The `back_log` variable controls the depth of this queue. If it fills up, new connections are refused with "Too many connections."

```sql
SHOW VARIABLES LIKE 'back_log';
SHOW STATUS LIKE 'Connection_errors_%';
```

## Monitoring Active Connections

```sql
-- List all current connections
SHOW PROCESSLIST;

-- More detail via performance_schema
SELECT thread_id, user, host, command, time, state
FROM performance_schema.threads
WHERE type = 'FOREGROUND';
```

## Summary

MySQL handles each client connection with a dedicated OS thread by default. The thread cache reduces the cost of repeated connect/disconnect cycles, while variables like `max_connections`, `thread_cache_size`, and `wait_timeout` govern behavior under load. Monitoring `Threads_running` vs. `Threads_connected` gives a quick signal of whether connections are queuing up, letting you tune proactively before exhaustion occurs.
