# How to Set Up Source-Replica Replication in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Source, Replica, High Availability, Database

Description: Learn how to configure MySQL source-replica (master-slave) replication step by step, including binary log setup, user creation, and replica sync.

---

## Overview

MySQL source-replica replication (formerly called master-slave) keeps a replica server synchronized with a source server by streaming and replaying binary log events. It is commonly used for read scaling, high availability, and offloading backups to the replica. This guide covers position-based replication, which works on MySQL 5.7 and 8.0.

## Architecture

```
[Source Server]  --binary log-->  [Replica Server]
  writes                           reads + replays
```

The replica runs two threads: an I/O thread that reads binary log events from the source, and an SQL thread that applies them to the replica's data.

## Step 1: Configure the Source Server

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf` (or `/etc/my.cnf`) on the source:

```ini
[mysqld]
server-id           = 1
log_bin             = /var/log/mysql/mysql-bin.log
binlog_expire_logs_seconds = 604800
max_binlog_size     = 100M
binlog_format       = ROW
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## Step 2: Create a Replication User on the Source

```sql
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'StrongReplPass!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
```

For tighter security, replace `'%'` with the replica's IP address.

## Step 3: Take a Snapshot of the Source

Lock the tables, record the binary log position, and take a dump:

```bash
# Connect to MySQL and lock tables
mysql -u root -p -e "FLUSH TABLES WITH READ LOCK;"

# In a second terminal, record the binary log position
mysql -u root -p -e "SHOW MASTER STATUS\G"
```

Note the `File` and `Position` values from the output, for example:
- File: `mysql-bin.000003`
- Position: `154`

Then create the snapshot:

```bash
mysqldump -u root -p --all-databases --single-transaction \
  --master-data=2 > source_snapshot.sql
```

`--master-data=2` embeds the binary log coordinates as a comment in the dump file. Unlock tables after the dump completes:

```sql
UNLOCK TABLES;
```

## Step 4: Configure the Replica Server

Edit `my.cnf` on the replica:

```ini
[mysqld]
server-id         = 2
relay-log         = /var/log/mysql/mysql-relay-bin.log
read_only         = ON
log_replica_updates = ON
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## Step 5: Import the Source Snapshot on the Replica

```bash
mysql -u root -p < source_snapshot.sql
```

## Step 6: Configure Replication on the Replica

```sql
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST     = '192.168.1.100',
    SOURCE_USER     = 'repl_user',
    SOURCE_PASSWORD = 'StrongReplPass!',
    SOURCE_LOG_FILE = 'mysql-bin.000003',
    SOURCE_LOG_POS  = 154;
```

In MySQL 5.7, the syntax uses the older `CHANGE MASTER TO` with `MASTER_HOST`, `MASTER_USER`, etc.

## Step 7: Start Replication and Verify

```sql
START REPLICA;
SHOW REPLICA STATUS\G
```

Check these fields:
- `Replica_IO_Running: Yes`
- `Replica_SQL_Running: Yes`
- `Seconds_Behind_Source: 0`

If either thread shows `No`, check `Last_IO_Error` and `Last_SQL_Error` for details.

## Monitoring Replication Lag

```sql
SELECT
    CHANNEL_NAME,
    SERVICE_STATE,
    LAST_ERROR_MESSAGE,
    LAST_HEARTBEAT_TIMESTAMP,
    TIME_SINCE_LAST_SOURCE_HEARTBEAT
FROM performance_schema.replication_connection_status;
```

## Skipping a Failed Transaction (Use with Caution)

```sql
STOP REPLICA SQL_THREAD;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;
START REPLICA SQL_THREAD;
```

Only skip errors you have analyzed and confirmed are safe to ignore.

## Summary

MySQL source-replica replication streams binary log events from the source to the replica for continuous synchronization. The setup involves enabling binary logging on the source, creating a replication user, taking a consistent snapshot, and pointing the replica at the source with `CHANGE REPLICATION SOURCE TO`. Monitor `SHOW REPLICA STATUS` for lag and errors after enabling replication.
