# How to Promote a Replica to Source in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Failover, High Availability, Administration

Description: Learn how to promote a MySQL replica to become the new source server during planned maintenance or emergency failover scenarios.

---

Promoting a replica to source is a critical operation performed during planned maintenance (switchover) or emergency failover when the primary server fails. The procedure differs slightly depending on whether you use GTID or position-based replication, and whether the promotion is planned or unplanned.

## Planned Switchover vs Emergency Failover

**Planned switchover**: The current source is healthy, and you are promoting a replica in a controlled manner for maintenance. Zero data loss is achievable.

**Emergency failover**: The current source is unavailable. You promote the most up-to-date replica, accepting possible data loss.

## Planned Switchover Procedure

**Step 1: Stop writes to the source**

```sql
-- On the source, prevent new writes
FLUSH TABLES WITH READ LOCK;
-- Or set the server read-only:
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

**Step 2: Wait for the replica to catch up**

```sql
-- On the replica
SHOW REPLICA STATUS\G
-- Wait for Seconds_Behind_Source = 0
```

**Step 3: Stop replication on the replica**

```sql
-- On the replica
STOP REPLICA;
RESET REPLICA ALL;
```

**Step 4: Promote the replica**

```sql
-- On the replica, make it writable
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;

-- If needed, enable binary logging (should already be ON)
SHOW VARIABLES LIKE 'log_bin';
```

**Step 5: Update application connection strings**

Point your application, load balancer, or ProxySQL to the new source.

**Step 6: Reconfigure the old source as a replica (optional)**

```sql
-- On the old source (now becoming a replica)
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'new-source.example.com',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplPassword',
  SOURCE_AUTO_POSITION = 1;

START REPLICA;
```

## Emergency Failover Procedure

When the source is down, identify the most up-to-date replica:

```sql
-- On each replica, check how far ahead it is
SHOW REPLICA STATUS\G
-- Look at: Exec_Master_Log_Pos (position-based) or gtid_executed (GTID)
```

With GTID, check the executed GTID set:

```sql
SELECT @@global.gtid_executed;
```

The replica with the largest GTID set or highest applied position is the best candidate.

**Promote the best replica:**

```sql
-- Stop replication
STOP REPLICA;
RESET REPLICA ALL;

-- Make writable
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
```

**Point remaining replicas to the new source:**

```sql
-- On other replicas
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'new-source.example.com',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplPassword',
  SOURCE_AUTO_POSITION = 1;

START REPLICA;
SHOW REPLICA STATUS\G
```

## Verifying the New Source

```sql
-- On new source
SHOW MASTER STATUS;
SHOW BINARY LOGS;

-- Create a test record
INSERT INTO mydb.promotion_test VALUES (NOW(), 'promoted');

-- Verify it appears on replicas
-- On replicas:
SELECT * FROM mydb.promotion_test;
```

## Updating DNS and Load Balancers

```bash
# Example: update Route 53 DNS entry
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123EXAMPLE \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "mysql.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "new-source.example.com"}]
      }
    }]
  }'
```

## Summary

For planned switchovers, set the source to read-only, wait for the replica to catch up (Seconds_Behind_Source = 0), stop replication, and make the replica writable. For emergency failovers, identify the most current replica by comparing `gtid_executed` sets, promote it by running `RESET REPLICA ALL` and disabling `read_only`, then repoint all other replicas and applications to the new source. Always use GTID replication to simplify repointing replicas after a failover.
