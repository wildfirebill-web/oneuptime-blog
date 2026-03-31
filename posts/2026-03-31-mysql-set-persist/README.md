# How to Use SET PERSIST in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SET PERSIST, Configuration, MySQL 8, Administration

Description: Master MySQL 8 SET PERSIST to apply and permanently save server variable changes, audit who changed what, and manage the mysqld-auto.cnf configuration file.

---

## SET PERSIST Overview

`SET PERSIST` is a MySQL 8.0 feature that applies a server variable change immediately (like `SET GLOBAL`) and writes the value to `mysqld-auto.cnf` in the data directory. The file is loaded at startup after `my.cnf`, so persisted values override file-based configuration.

This eliminates the manual step of updating `my.cnf` after every runtime configuration change.

## Basic Usage

```sql
-- Apply immediately and persist across restarts
SET PERSIST max_connections = 500;
SET PERSIST wait_timeout = 1800;
SET PERSIST slow_query_log = ON;
SET PERSIST long_query_time = 1.0;
```

## Persist Multiple Variables

```sql
SET PERSIST
  tmp_table_size = 67108864,
  max_heap_table_size = 67108864,
  join_buffer_size = 4194304,
  sort_buffer_size = 4194304;
```

## View Persisted Values

```sql
SELECT VARIABLE_NAME, SET_VALUE
FROM performance_schema.persisted_variables
ORDER BY VARIABLE_NAME;
```

```text
+---------------------------+-----------+
| VARIABLE_NAME             | SET_VALUE |
+---------------------------+-----------+
| join_buffer_size          | 4194304   |
| long_query_time           | 1         |
| max_connections           | 500       |
| slow_query_log            | ON        |
| sort_buffer_size          | 4194304   |
| tmp_table_size            | 67108864  |
| wait_timeout              | 1800      |
+---------------------------+-----------+
```

## Audit Who Persisted Each Value

MySQL records metadata about each persisted change:

```sql
SELECT
  VARIABLE_NAME,
  VARIABLE_VALUE,
  VARIABLE_SOURCE,
  SET_USER,
  SET_HOST,
  SET_TIME
FROM performance_schema.variables_info
WHERE VARIABLE_SOURCE = 'PERSISTED'
ORDER BY SET_TIME DESC;
```

## Difference Between SET GLOBAL and SET PERSIST

| Feature | SET GLOBAL | SET PERSIST |
|---------|-----------|-------------|
| Immediate effect | Yes | Yes |
| Survives restart | No | Yes |
| Writes to file | No | mysqld-auto.cnf |
| Audit trail | No | Yes (metadata) |

## Use Case: Enable Slow Query Log Without Restart

```sql
-- Enable slow query log dynamically and permanently
SET PERSIST slow_query_log = ON;
SET PERSIST slow_query_log_file = '/var/log/mysql/slow-query.log';
SET PERSIST long_query_time = 2;
SET PERSIST log_queries_not_using_indexes = ON;
```

Verify:

```sql
SHOW GLOBAL VARIABLES LIKE 'slow%';
```

## Remove a Persisted Value

To stop using a persisted value and revert to `my.cnf` (or the compiled default):

```sql
RESET PERSIST max_connections;
```

After the next restart, `max_connections` will be read from `my.cnf` or use the default.

To remove all persisted values:

```sql
RESET PERSIST;
```

## Disable SET PERSIST

If you want to prevent any changes to `mysqld-auto.cnf` (for example, in production where all config is managed via `my.cnf`):

```ini
# my.cnf
[mysqld]
persisted_globals_load = OFF
```

With this setting, `mysqld-auto.cnf` is ignored at startup and `SET PERSIST` still writes to the file but the values are not loaded.

## Summary

`SET PERSIST` in MySQL 8 is the recommended way to make server variable changes that survive restarts. It applies changes immediately and writes to `mysqld-auto.cnf`. Audit persisted changes in `performance_schema.variables_info`. Remove individual persisted values with `RESET PERSIST variable_name` when you want `my.cnf` to take precedence. Disable persistence loading with `persisted_globals_load = OFF` for config-file-managed environments.
