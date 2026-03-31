# How to Set Up GTID-Based Replication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Replication, Gtid, High Availability, Administration

Description: Learn how to configure GTID-based replication in MySQL, which simplifies failover and replica promotion by using global transaction identifiers instead of binary log positions.

---

## What Is GTID-Based Replication?

Global Transaction Identifiers (GTIDs) are unique identifiers assigned to every committed transaction on the source. A GTID consists of the server UUID and a transaction sequence number: `server_uuid:transaction_id`.

GTID-based replication eliminates the need to track binary log file names and positions. Replicas automatically find where to resume replication after failover, making it the preferred approach for modern MySQL deployments.

## GTID Format

```text
3E11FA47-71CA-11E1-9E33-C80AA9429562:1-47
```

- `3E11FA47-71CA-11E1-9E33-C80AA9429562` is the source server's UUID
- `1-47` means transactions 1 through 47 from that server

## Step 1 - Configure the Source Server

```text
[mysqld]
server_id = 1
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

Key variables:
- `gtid_mode = ON` - enables GTIDs
- `enforce_gtid_consistency = ON` - prevents statements that cannot be logged with GTIDs (required when `gtid_mode = ON`)

Restart the source:

```bash
sudo systemctl restart mysql
```

## Step 2 - Configure the Replica Server

```text
[mysqld]
server_id = 2
relay_log = /var/log/mysql/relay-bin
log_bin = /var/log/mysql/mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON
log_replica_updates = ON
```

Restart the replica.

## Step 3 - Create a Replication User on the Source

```sql
CREATE USER 'repl_user'@'10.0.1.%' IDENTIFIED BY 'ReplPassword!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'10.0.1.%';
```

## Step 4 - Take a Snapshot with GTID Information

```bash
mysqldump \
  --single-transaction \
  --source-data=2 \
  --set-gtid-purged=ON \
  --all-databases \
  --routines \
  --triggers \
  --events \
  -u root -p > /tmp/source_gtid.sql
```

`--set-gtid-purged=ON` includes a `SET @@GLOBAL.gtid_purged` statement in the dump, which tells the replica which GTIDs have already been executed.

## Step 5 - Restore on the Replica

```bash
mysql -u root -p < /tmp/source_gtid.sql
```

## Step 6 - Connect the Replica Using GTID Auto-Positioning

With GTIDs, no log file or position is needed:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.1',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplPassword!',
  SOURCE_AUTO_POSITION = 1;
```

`SOURCE_AUTO_POSITION = 1` instructs MySQL to use GTID auto-positioning instead of binary log coordinates.

For MySQL 5.7 / earlier 8.0:

```sql
CHANGE MASTER TO
  MASTER_HOST = '10.0.1.1',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'ReplPassword!',
  MASTER_AUTO_POSITION = 1;
```

## Step 7 - Start Replication and Verify

```sql
START REPLICA;

SHOW REPLICA STATUS\G
```

Look for:
```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Retrieved_Gtid_Set: 3E11FA47-...:1-150
Executed_Gtid_Set: 3E11FA47-...:1-150
```

When `Retrieved_Gtid_Set` equals `Executed_Gtid_Set`, the replica is fully caught up.

## Checking GTID State

```sql
-- GTIDs executed on this server
SELECT @@global.gtid_executed;

-- GTIDs purged from binary logs
SELECT @@global.gtid_purged;
```

## Failover with GTIDs

Promoting a replica to source is simpler with GTIDs:

```sql
-- On the new source (former replica):
STOP REPLICA;
RESET REPLICA ALL;

-- Point other replicas to the new source
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.2',
  SOURCE_AUTO_POSITION = 1;
```

GTIDs ensure the replicas know exactly which transactions they have and which they need, preventing duplicate execution.

## Statements Incompatible with GTID Mode

`enforce_gtid_consistency = ON` blocks:
- `CREATE TABLE ... SELECT`
- `CREATE TEMPORARY TABLE` inside transactions
- Non-transactional DML inside a transaction affecting transactional tables

If you see:

```text
ERROR 1786 (HY000): Statement violates GTID consistency
```

Rewrite the statement to be GTID-compatible.

## Summary

GTID-based replication simplifies MySQL replication by replacing binary log coordinates with globally unique transaction IDs. Enable `gtid_mode = ON` and `enforce_gtid_consistency = ON` on all servers, use `SOURCE_AUTO_POSITION = 1` in `CHANGE REPLICATION SOURCE TO`, and monitor with `SHOW REPLICA STATUS`. GTIDs make failover and replica re-pointing straightforward compared to position-based replication.
