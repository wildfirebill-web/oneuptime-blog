# How to Set Up Source-Replica Replication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Source-Replica, High Availability, Administration

Description: Learn how to configure MySQL source-replica (master-slave) replication step by step, including binary log setup, snapshot transfer, and replica configuration.

---

## What Is Source-Replica Replication?

In MySQL replication, the source (formerly called master) logs all data changes to its binary log. The replica (formerly called slave) reads those binary log events and applies them to keep its data synchronized with the source. This enables:

- **Read scaling** - route read queries to replicas
- **High availability** - promote a replica if the source fails
- **Backups** - take backups from replicas without affecting the source

## Step 1 - Configure the Source Server

Add to `/etc/mysql/my.cnf` on the source:

```text
[mysqld]
server_id = 1
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
binlog_row_image = MINIMAL
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
```

Restart the source:

```bash
sudo systemctl restart mysql
```

## Step 2 - Create a Replication User on the Source

```sql
CREATE USER 'repl_user'@'10.0.1.%' IDENTIFIED BY 'ReplPassword!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'10.0.1.%';
FLUSH PRIVILEGES;
```

## Step 3 - Take a Consistent Snapshot of the Source

### Using mysqldump

```bash
mysqldump \
  --single-transaction \
  --master-data=2 \
  --all-databases \
  --routines \
  --triggers \
  --events \
  -u root -p > /tmp/source_snapshot.sql
```

`--master-data=2` adds the binary log file and position as a comment in the dump. Note the position from the dump file:

```bash
grep "CHANGE MASTER TO" /tmp/source_snapshot.sql | head -1
```

```text
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=1234;
```

### Alternative - Using FLUSH TABLES WITH READ LOCK

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Note the `File` and `Position` values, then export and `UNLOCK TABLES`.

## Step 4 - Configure the Replica Server

Add to `/etc/mysql/my.cnf` on the replica:

```text
[mysqld]
server_id = 2
relay_log = /var/log/mysql/relay-bin
read_only = ON
log_bin = /var/log/mysql/mysql-bin
log_replica_updates = ON
```

- `read_only = ON` - prevents direct writes to the replica (use `super_read_only` for stricter enforcement).
- `log_replica_updates = ON` - replica logs received events to its own binlog (needed for chained replication).

Restart the replica:

```bash
sudo systemctl restart mysql
```

## Step 5 - Restore the Snapshot on the Replica

```bash
mysql -u root -p < /tmp/source_snapshot.sql
```

## Step 6 - Point the Replica to the Source

For MySQL 8.0.23+:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.1',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplPassword!',
  SOURCE_LOG_FILE = 'mysql-bin.000003',
  SOURCE_LOG_POS = 1234,
  SOURCE_SSL = 1;
```

For MySQL 5.7 / earlier 8.0:

```sql
CHANGE MASTER TO
  MASTER_HOST = '10.0.1.1',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'ReplPassword!',
  MASTER_LOG_FILE = 'mysql-bin.000003',
  MASTER_LOG_POS = 1234;
```

## Step 7 - Start Replication

```sql
START REPLICA;
-- or for older versions:
START SLAVE;
```

## Step 8 - Verify Replication Status

```sql
SHOW REPLICA STATUS\G
-- or:
SHOW SLAVE STATUS\G
```

Look for:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
```

`Seconds_Behind_Source: 0` means the replica is fully caught up.

## Monitoring Replication Lag

```sql
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  LAST_ERROR_MESSAGE,
  LAST_ERROR_TIMESTAMP
FROM performance_schema.replication_applier_status_by_worker;
```

## Summary

MySQL source-replica replication uses the binary log to synchronize changes from the source to one or more replicas. Configure `server_id` and `log_bin` on both servers, create a replication user, take a consistent snapshot with `--master-data=2`, restore it on the replica, and use `CHANGE REPLICATION SOURCE TO` to point the replica at the source. Verify with `SHOW REPLICA STATUS` and monitor `Seconds_Behind_Source`.
