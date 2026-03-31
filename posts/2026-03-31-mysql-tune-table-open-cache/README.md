# How to Tune table_open_cache for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, Tuning, Cache, Table

Description: Tune MySQL table_open_cache to reduce file descriptor overhead by keeping frequently accessed table file descriptors open and ready for reuse.

---

Every time MySQL accesses a table, it needs to open the table's files - the .ibd data file for InnoDB tables. Opening files has OS overhead. The `table_open_cache` controls how many open table file descriptors MySQL keeps cached. When a query needs a table that is already in the cache, it skips the open() system call entirely.

## How the Table Open Cache Works

MySQL maintains a least-recently-used (LRU) cache of open table file descriptors. Each entry represents one table opened by one thread. With `table_open_cache = 2000` and 200 concurrent connections, the cache can hold up to 2000 open table descriptors across all connections.

Check current settings and hit rates:

```sql
SHOW VARIABLES LIKE 'table_open_cache';
SHOW GLOBAL STATUS LIKE 'Open%';
```

```text
Open_files          358
Open_table_definitions 892
Open_tables         1024
Opened_tables       48291
```

The key metric is `Opened_tables`. If it grows rapidly relative to uptime, tables are not being found in the cache and are being reopened frequently.

## Calculating the Cache Hit Rate

```sql
SELECT
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Open_tables') AS open_tables,
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Opened_tables') AS opened_tables,
  ROUND(
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Open_tables') /
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Opened_tables') * 100,
    2
  ) AS cache_utilization_pct;
```

Compare `Open_tables` to `table_open_cache`:

```sql
SELECT
  @@table_open_cache AS configured_cache,
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Open_tables') AS currently_open;
```

If `Open_tables` is consistently equal to `table_open_cache`, the cache is full and tables are being evicted. Increase the cache size.

## Setting table_open_cache

The default in MySQL 8.0 is 4000. For databases with many tables, increase this proportionally:

```sql
-- Count your tables to estimate the required cache size
SELECT COUNT(*) AS total_tables FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys');

-- Set cache size
SET GLOBAL table_open_cache = 8000;
```

In configuration:

```text
[mysqld]
table_open_cache = 8000
table_open_cache_instances = 16
```

The `table_open_cache_instances` setting divides the cache into shards, reducing mutex contention under high concurrency.

## File Descriptor Limits

Each open table consumes a file descriptor. Ensure the OS file descriptor limit is high enough:

```bash
# Check current limit
ulimit -n

# Check MySQL's open files limit
cat /proc/$(pidof mysqld)/limits | grep "open files"

# Increase in /etc/security/limits.conf
mysql soft nofile 65535
mysql hard nofile 65535

# Or in systemd service
sudo systemctl edit mysql
```

```text
[Service]
LimitNOFILE=65535
```

Verify MySQL can actually open enough files:

```sql
SHOW VARIABLES LIKE 'open_files_limit';
```

## table_definition_cache

Related to `table_open_cache`, the `table_definition_cache` controls how many `.frm`-equivalent table definitions are kept in memory:

```sql
SHOW VARIABLES LIKE 'table_definition_cache';

-- Set to at least the number of tables in your database
SET GLOBAL table_definition_cache = 4000;
```

## Summary

Tune `table_open_cache` when `Opened_tables` grows rapidly or `Open_tables` consistently equals the configured cache size. Set the cache to at least the total number of tables accessed by your typical query workload, and increase `table_open_cache_instances` to 8-16 for high-concurrency servers to reduce lock contention. Always check the OS file descriptor limit alongside `open_files_limit` to ensure MySQL can actually open the number of files implied by your cache configuration.
