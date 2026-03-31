# How to Use START REPLICA and STOP REPLICA Statements in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Administration

Description: Learn how to use START REPLICA and STOP REPLICA in MySQL 8 to control replication threads, manage channels, and handle replication errors.

---

## Overview of START REPLICA and STOP REPLICA

`START REPLICA` and `STOP REPLICA` control the replication threads on a MySQL replica server. They were introduced in MySQL 8.0.22 as replacements for `START SLAVE` and `STOP SLAVE`. These commands manage the I/O thread (which fetches binary log events from the source) and the SQL thread (which applies those events to the replica's data).

```sql
-- Start all replica threads
START REPLICA;

-- Stop all replica threads
STOP REPLICA;
```

## Starting and Stopping Individual Threads

You can start or stop threads independently using the `IO_THREAD` and `SQL_THREAD` options:

```sql
-- Stop only the I/O thread (stop receiving new events)
STOP REPLICA IO_THREAD;

-- Stop only the SQL thread (stop applying events, keep receiving)
STOP REPLICA SQL_THREAD;

-- Start only the I/O thread
START REPLICA IO_THREAD;

-- Start only the SQL thread
START REPLICA SQL_THREAD;
```

Stopping the SQL thread while keeping the I/O thread running is useful when you want to pause applying events temporarily (for example, to examine the relay log) while still fetching them.

## Checking Replication Status

After starting or stopping replication, always check the status:

```sql
SHOW REPLICA STATUS\G
```

Key fields:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_IO_Error:
Last_SQL_Error:
```

If either thread shows `No`, look at the corresponding error field for the cause.

## Skipping a Failed Transaction

When a SQL thread stops due to an error (such as a duplicate key or constraint violation), you can skip the problematic transaction:

```sql
-- Stop the SQL thread
STOP REPLICA SQL_THREAD;

-- Skip one event (binary log position-based replication)
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;

-- Restart the SQL thread
START REPLICA SQL_THREAD;
```

For GTID-based replication, skip the transaction by injecting an empty transaction:

```sql
STOP REPLICA SQL_THREAD;

-- Get the GTID of the failing transaction from SHOW REPLICA STATUS
SET GTID_NEXT = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:12345';
BEGIN; COMMIT;  -- empty transaction
SET GTID_NEXT = 'AUTOMATIC';

START REPLICA SQL_THREAD;
```

## Managing Replication Channels

For multi-source replication, you control threads per channel:

```sql
-- Start a specific channel
START REPLICA FOR CHANNEL 'source_db1';

-- Stop a specific channel
STOP REPLICA FOR CHANNEL 'source_db1';

-- Start all channels
START REPLICA;

-- Check status per channel
SHOW REPLICA STATUS FOR CHANNEL 'source_db1'\G
```

## UNTIL Clause

`START REPLICA` supports an `UNTIL` clause to stop applying events at a specific binary log position or GTID:

```sql
-- Stop SQL thread when it reaches this position
START REPLICA SQL_THREAD UNTIL
  SOURCE_LOG_FILE = 'mysql-bin.000010',
  SOURCE_LOG_POS = 5000;

-- Stop at a specific GTID
START REPLICA SQL_THREAD UNTIL SQL_AFTER_GTIDS = 'uuid:1-100';
```

This is useful for replaying events up to a specific point in time for recovery purposes.

## Graceful Shutdown Workflow

Before shutting down a replica cleanly:

```sql
-- Stop replication gracefully
STOP REPLICA;

-- Verify both threads have stopped
SHOW REPLICA STATUS\G

-- Now safe to shut down MySQL
SHUTDOWN;
```

## Required Privileges

```sql
-- The REPLICATION_SLAVE_ADMIN privilege is needed
GRANT REPLICATION_SLAVE_ADMIN ON *.* TO 'ops_user'@'localhost';
```

## Summary

`START REPLICA` and `STOP REPLICA` are the primary commands for managing replication threads in MySQL 8. Use thread-specific options to control I/O and SQL threads independently, leverage the `UNTIL` clause for controlled point-in-time replay, and manage individual channels in multi-source setups. Always verify replication health with `SHOW REPLICA STATUS` after any thread state change.
