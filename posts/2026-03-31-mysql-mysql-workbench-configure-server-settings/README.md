# How to Configure Server Settings in MySQL Workbench

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Configuration, Server, Setting

Description: Learn how to use MySQL Workbench to view and modify MySQL server configuration variables, edit the options file, and apply runtime changes graphically.

---

## Introduction

MySQL Workbench provides two complementary interfaces for managing MySQL server configuration: the **Options File** editor for persistent configuration changes (my.cnf / my.ini), and the **Status and System Variables** panel for viewing and modifying dynamic variables at runtime. Together they replace manual editing of configuration files for most common settings.

## Accessing Configuration Tools

Connect to a MySQL server in Workbench and navigate to:

```text
Navigator > Instance > Options File
```

Or for runtime variables:

```text
Server > Status and System Variables
```

## Options File Editor

The Options File editor reads and writes the MySQL configuration file (`my.cnf` on Linux/macOS, `my.ini` on Windows). Settings are organized into tabs by category:

### General Tab

Key settings visible here:

```ini
[mysqld]
datadir = /var/lib/mysql
socket  = /var/lib/mysql/mysql.sock
port    = 3306
```

### InnoDB Tab

```ini
[mysqld]
innodb_buffer_pool_size   = 1G
innodb_log_file_size      = 256M
innodb_flush_log_at_trx_commit = 1
innodb_file_per_table     = ON
```

Change `innodb_buffer_pool_size` to approximately 70-80% of available RAM for dedicated MySQL servers.

### Logging Tab

```ini
[mysqld]
general_log             = OFF
slow_query_log          = ON
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 1
log_output              = FILE
```

### Networking Tab

```ini
[mysqld]
bind-address      = 0.0.0.0
max_connections   = 200
connect_timeout   = 10
wait_timeout      = 28800
```

After editing, click **Apply** and restart MySQL for non-dynamic settings to take effect.

## Status and System Variables

For runtime-adjustable variables, use:

```text
Server > Status and System Variables
```

Filter the variable list by typing in the search box:

```text
Search: innodb_buffer_pool
```

Results show:
- Variable name
- Current value
- Scope (Global / Session / Both)
- Type (boolean, integer, string)

## Modifying Dynamic Variables

Some variables can be changed without a restart. Click on a variable and edit its value, then click **Apply**. Workbench executes:

```sql
SET GLOBAL max_connections = 500;
```

To also persist the change across restarts (MySQL 8.0+):

```sql
SET PERSIST max_connections = 500;
```

Or via Workbench, check the **Persist** checkbox before applying.

## Common Variables to Tune

```sql
-- Buffer pool (memory for InnoDB data and indexes)
SET PERSIST innodb_buffer_pool_size = 4294967296; -- 4GB

-- Max connections
SET PERSIST max_connections = 300;

-- Query cache (MySQL 5.7 only - removed in 8.0)
-- SET GLOBAL query_cache_size = 0;

-- Slow query threshold
SET PERSIST long_query_time = 0.5;

-- Temporary table size
SET PERSIST tmp_table_size = 67108864;    -- 64MB
SET PERSIST max_heap_table_size = 67108864;
```

## Viewing Current Status Variables

Status variables (not configurable, only observable) show server activity:

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
```

In Workbench, switch to the **Status Variables** tab to browse these graphically.

## Summary

MySQL Workbench's Options File editor and Status and System Variables panel give you graphical control over MySQL server configuration. Use the Options File editor for persistent changes requiring a restart, and the Status Variables panel for real-time monitoring and dynamic variable adjustments. Always test configuration changes in a non-production environment before applying to production.
