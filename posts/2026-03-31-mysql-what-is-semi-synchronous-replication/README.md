# What Is Semi-Synchronous Replication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Semi-Synchronous, Durability, High Availability

Description: Semi-synchronous replication in MySQL requires at least one replica to acknowledge receipt of a transaction before the source commits, reducing the risk of data loss.

---

## Overview

MySQL offers three replication durability modes: asynchronous, semi-synchronous, and synchronous (via Group Replication). Semi-synchronous replication sits between the two extremes: the source waits for at least one replica to confirm it has received and written the transaction to its relay log before returning success to the client. The replica does not need to fully apply the transaction -- only to acknowledge receipt.

This provides much stronger durability guarantees than asynchronous replication while avoiding the full performance overhead of fully synchronous replication.

## Installing the Plugin

Semi-synchronous replication is delivered as a plugin that must be installed on both source and replicas:

```sql
-- On the source
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = ON;

-- On each replica
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = ON;
```

To make it persistent across restarts, add to `my.cnf`:

```ini
[mysqld]
plugin-load-add = semisync_source.so
rpl_semi_sync_source_enabled = 1
rpl_semi_sync_source_timeout = 10000
```

## How the Commit Flow Works

In standard asynchronous replication:
1. Source writes to binary log
2. Source commits and returns to client
3. Replica reads and applies (later, asynchronously)

In semi-synchronous replication:
1. Source writes to binary log
2. Source sends the event to replicas
3. At least one replica writes to relay log and sends acknowledgment
4. Source commits and returns to client

The source waits for acknowledgment up to `rpl_semi_sync_source_timeout` milliseconds. If no replica acknowledges in time, the source falls back to asynchronous mode automatically.

## Configuring the Timeout

The timeout controls how long the source waits before falling back:

```sql
-- Wait up to 10 seconds for replica acknowledgment
SET GLOBAL rpl_semi_sync_source_timeout = 10000;

-- Require acknowledgment from at least N replicas (MySQL 5.7+)
SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;
```

## Monitoring Semi-Synchronous Status

```sql
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

Key status variables:
- `Rpl_semi_sync_source_status`: whether semi-sync is active (`ON`) or fell back to async (`OFF`)
- `Rpl_semi_sync_source_yes_tx`: transactions committed with semi-sync acknowledgment
- `Rpl_semi_sync_source_no_tx`: transactions that timed out and fell back to async

## After-Commit vs After-Sync

MySQL 5.7 introduced `rpl_semi_sync_source_wait_point`:
- `AFTER_COMMIT` (default pre-5.7): source commits before waiting for replica ack; a failover could return committed data not yet on any replica
- `AFTER_SYNC` (default 5.7+): source waits for replica ack before committing; no phantom reads possible after failover

```sql
SET GLOBAL rpl_semi_sync_source_wait_point = 'AFTER_SYNC';
```

## Summary

Semi-synchronous replication improves data durability over asynchronous replication by requiring at least one replica to acknowledge each transaction before the source confirms the commit. It automatically degrades to asynchronous mode if replicas are unavailable, balancing safety with availability. It is a practical choice for environments where some replication lag is acceptable but data loss on failover is not.
