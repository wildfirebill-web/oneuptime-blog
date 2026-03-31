# How to Replicate Data from MySQL to ClickHouse in Real-Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MySQL, Replication, CDC, Real-Time Analytics

Description: Replicate MySQL tables to ClickHouse in real-time using the MaterializedMySQL database engine for live analytical queries on operational data.

---

## Overview

ClickHouse's `MaterializedMySQL` database engine replicates data from a MySQL instance using MySQL's binary log (binlog). Once configured, ClickHouse continuously applies INSERT, UPDATE, and DELETE operations, keeping a local copy of MySQL tables that you can query at analytical speeds.

## Prerequisites

- MySQL 5.7 or 8.0 with binlog enabled
- ClickHouse 21.1 or later
- A MySQL user with replication privileges

## Configure MySQL

Enable row-based binary logging in MySQL:

```text
[mysqld]
server-id         = 1
log_bin           = /var/log/mysql/mysql-bin.log
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 7
```

Create a replication user:

```sql
CREATE USER 'ch_replicator'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'ch_replicator'@'%';
FLUSH PRIVILEGES;
```

## Create the MaterializedMySQL Database

On the ClickHouse side:

```sql
CREATE DATABASE mysql_replica
ENGINE = MaterializedMySQL(
    'mysql-host:3306',
    'source_db',
    'ch_replicator',
    'strong_password'
)
SETTINGS
    max_rows_in_buffer = 65536,
    max_bytes_in_buffer = 1048576;
```

ClickHouse connects to MySQL, takes a snapshot of all tables, then tails the binlog for changes.

## Query Replicated Tables

Once replication starts, query MySQL tables directly from ClickHouse:

```sql
-- In ClickHouse
SELECT
    customer_id,
    sum(total_amount) AS lifetime_value
FROM mysql_replica.orders
WHERE status = 'completed'
GROUP BY customer_id
ORDER BY lifetime_value DESC
LIMIT 10;
```

## Monitor Replication Lag

```sql
SELECT
    database,
    table,
    elapsed,
    read_rows,
    written_rows
FROM system.replicas
WHERE database = 'mysql_replica';
```

Also check the binlog position:

```sql
SELECT * FROM system.databases
WHERE engine = 'MaterializedMySQL';
```

## Handle Data Type Differences

ClickHouse maps MySQL types automatically:

```text
INT          -> Int32
BIGINT       -> Int64
VARCHAR(n)   -> String
DATETIME     -> DateTime
DECIMAL(p,s) -> Decimal(p, s)
TINYINT(1)   -> Bool
```

For columns that do not map cleanly, add a materialized view to transform them:

```sql
CREATE MATERIALIZED VIEW orders_clean
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY order_id AS
SELECT
    order_id,
    customer_id,
    toDecimal64(total_amount, 2) AS amount,
    toDateTime(created_at)       AS created_at,
    toDateTime(updated_at)       AS updated_at
FROM mysql_replica.orders;
```

## Pause and Resume Replication

```sql
-- Stop syncing
SYSTEM STOP FETCHES mysql_replica;

-- Resume syncing
SYSTEM START FETCHES mysql_replica;
```

## Summary

The `MaterializedMySQL` engine gives ClickHouse a live read replica of any MySQL database with sub-second replication lag. Use it to run analytical queries against your operational MySQL data without adding load to the primary server.
