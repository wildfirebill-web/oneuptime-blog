# How to Tune max_connections for Your MySQL Workload

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Tuning, Connection, Configuration, Database

Description: Learn how to correctly size max_connections in MySQL to prevent 'Too many connections' errors while avoiding excessive memory consumption.

---

## Why max_connections Matters

`max_connections` sets the maximum number of simultaneous client connections MySQL accepts. Set it too low and applications get "Too many connections" errors. Set it too high and MySQL runs out of memory because each connection consumes per-thread memory (sort buffer, read buffer, join buffer, etc.).

Getting this value right requires understanding your workload and available memory.

## Check the Current Setting and Usage

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Connection_errors_max_connections';
```

```text
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| max_connections                  | 151   |
| Max_used_connections             | 87    |
| Threads_connected                | 42    |
| Connection_errors_max_connections| 0     |
+----------------------------------+-------+
```

`Max_used_connections` shows the highest watermark since the server started. `Connection_errors_max_connections` shows how many connections were rejected.

## Calculate Per-Connection Memory Usage

Each connection allocates several buffers when needed. Estimate the worst-case per-connection memory:

```sql
SELECT
  ROUND((
    @@read_buffer_size
    + @@read_rnd_buffer_size
    + @@sort_buffer_size
    + @@join_buffer_size
    + @@binlog_cache_size
    + @@thread_stack
  ) / 1024 / 1024, 2) AS per_connection_mb;
```

Typical defaults produce about 2-4 MB per connection. With custom buffer sizes, this can reach 10-20 MB.

## Calculate the Safe Maximum

```bash
# Total RAM: 16 GB
# Reserved for buffer pool: 12 GB
# Reserved for OS: 1 GB
# Available for connections: 3 GB

# At 4 MB per connection:
# max_connections = 3 * 1024 / 4 = 768
```

A conservative formula:

```text
max_connections = (available_RAM_MB) / per_connection_memory_MB
```

## Set max_connections

In `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```text
[mysqld]
max_connections = 500
```

Or set dynamically without restart:

```sql
SET GLOBAL max_connections = 500;
```

## The Role of Connection Pooling

`max_connections` should not be large just because you have many application instances. Connection pools (PgBouncer equivalent for MySQL: ProxySQL, MySQL Router) manage connection reuse at the application layer:

```text
Application Instances (1000 threads)
        |
    ProxySQL (50 persistent connections to MySQL)
        |
    MySQL (max_connections = 100)
```

With a connection pool, MySQL may only need 50-200 max connections even when serving thousands of application threads.

## Monitor Thread States

```sql
SHOW PROCESSLIST;
```

Or for aggregated stats:

```sql
SELECT command, count(*) as total
FROM information_schema.processlist
GROUP BY command;
```

```text
+---------+-------+
| command | total |
+---------+-------+
| Sleep   | 45    |
| Query   | 8     |
| Connect | 0     |
+---------+-------+
```

If most connections are in `Sleep` state, they are idle and consuming memory. This indicates you need a connection pool, not a higher `max_connections`.

## Set innodb_thread_concurrency to Limit Active Threads

Even with 500 allowed connections, only a fraction can execute InnoDB operations concurrently without degrading performance:

```text
[mysqld]
innodb_thread_concurrency = 0   # Let InnoDB manage automatically (recommended for MySQL 8.0)
```

## Key Settings That Affect Per-Thread Memory

```text
[mysqld]
sort_buffer_size = 256K       # Default 256K; increase only if you see sort_merge_passes
read_buffer_size = 128K       # Default 128K
read_rnd_buffer_size = 256K   # Used for ORDER BY on full table scans
join_buffer_size = 256K       # Default 256K; increase for hash joins
thread_stack = 1M             # Default 1M; rarely needs changing
```

Avoid setting these to large values globally - large per-thread buffers multiply by connection count.

## Summary

Set `max_connections` based on available memory minus buffer pool and OS overhead, not based on the number of application threads. Use connection pooling (ProxySQL or MySQL Router) to decouple application connection counts from MySQL connections. Monitor `Max_used_connections` and `Connection_errors_max_connections` to tune the value over time.
