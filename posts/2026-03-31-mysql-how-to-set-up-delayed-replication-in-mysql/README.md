# How to Set Up Delayed Replication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Delayed Replication, Disaster Recovery

Description: Learn how to configure MySQL delayed replication to intentionally lag a replica behind the primary as a safety net against accidental data loss.

---

## What Is Delayed Replication

Delayed replication configures a replica to apply transactions only after a specified time delay. If a destructive operation (like a mistaken `DROP TABLE`) is executed on the primary, you have a window to stop the replica before it applies the change, then recover the data from it.

The delay is set in seconds using the `SOURCE_DELAY` option.

## Prerequisites

- MySQL 5.6 or later
- A working primary-replica replication setup
- The replica should already be connected to the primary

## Step 1 - Stop the Replica

Before changing the delay, stop replication on the replica:

```sql
STOP REPLICA;
```

## Step 2 - Configure the Delay

Set the desired delay in seconds. The example below sets a 1-hour delay (3600 seconds):

```sql
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;
```

For older MySQL versions using deprecated syntax:

```sql
CHANGE MASTER TO MASTER_DELAY = 3600;
```

## Step 3 - Start the Replica

```sql
START REPLICA;
```

## Step 4 - Verify the Delay

Check that the delay is applied:

```sql
SHOW REPLICA STATUS\G
```

Look for:

```text
SQL_Delay: 3600
SQL_Remaining_Delay: 3587
```

`SQL_Remaining_Delay` shows the seconds until the next queued transaction will be applied.

## How to Use Delayed Replication for Recovery

If you accidentally drop or truncate a table on the primary, follow these steps:

**1. Stop the replica SQL thread immediately:**

```sql
STOP REPLICA SQL_THREAD;
```

**2. Check what transactions are pending:**

```sql
SHOW REPLICA STATUS\G
```

Note `Relay_Log_File` and `Relay_Log_Pos`.

**3. Use `mysqlbinlog` to inspect or roll forward to a safe point:**

```bash
mysqlbinlog --start-position=4 /var/lib/mysql/relay-bin.000010 | less
```

**4. Start the replica until just before the bad event:**

```sql
START REPLICA SQL_THREAD UNTIL
  RELAY_LOG_FILE = 'relay-bin.000010',
  RELAY_LOG_POS  = 2048;
```

**5. Export the affected table from the replica:**

```bash
mysqldump -u root -p mydb lost_table > lost_table.sql
```

**6. Import it back into the primary:**

```bash
mysql -u root -p mydb < lost_table.sql
```

## Removing the Delay

To revert to normal (zero-delay) replication:

```sql
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 0;
START REPLICA;
```

## Choosing a Delay Value

| Risk Tolerance | Delay Suggestion |
|---|---|
| Low (quick detection) | 1800 seconds (30 min) |
| Medium | 3600 seconds (1 hour) |
| High (larger window) | 86400 seconds (1 day) |

Choose based on how quickly your team can detect and respond to incidents.

## Summary

Delayed replication is a low-cost safety mechanism that creates a time window to recover from accidental data modifications. You configure it with `CHANGE REPLICATION SOURCE TO SOURCE_DELAY = N`, and use `STOP REPLICA SQL_THREAD` immediately after discovering an issue to halt the replica before the bad transaction applies.
