# How to Handle Replication Errors in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Error, Troubleshooting, Database

Description: Learn how to identify, diagnose, and resolve MySQL replication errors including SQL thread failures and data consistency issues.

---

Replication errors stop the replica SQL thread from applying events. The replica falls behind, data diverges, and applications relying on the replica receive stale or incorrect data. Knowing how to quickly identify the error type and apply the correct fix is an essential skill for MySQL administrators.

## Checking Replication Status

The first step is always to check the error details:

```sql
SHOW REPLICA STATUS\G
```

Key fields to examine:

```text
Replica_SQL_Running: No           -- SQL thread has stopped
Last_SQL_Error: Duplicate entry '42' for key 'PRIMARY'
Last_SQL_Errno: 1062
Relay_Log_File: relay-bin.000012
Relay_Log_Pos: 4521
```

Also check the replica's error log:

```bash
tail -100 /var/log/mysql/mysql-error.log | grep -i "replica\|error\|replication"
```

## Common Error Types

### Error 1062: Duplicate Entry

This happens when the replica already has a row that the source is trying to insert:

```text
Last_SQL_Error: Could not execute Write_rows event on table mydb.orders;
Duplicate entry '42' for key 'orders.PRIMARY', Error_code: 1062
```

Fix by deleting the conflicting row and restarting the replica:

```sql
STOP REPLICA SQL_THREAD;

-- Delete the duplicate row
DELETE FROM mydb.orders WHERE id = 42;

START REPLICA SQL_THREAD;
SHOW REPLICA STATUS\G
```

### Error 1032: Row Not Found

The replica cannot find a row to update or delete:

```text
Last_SQL_Error: Could not execute Update_rows event on table mydb.orders;
Can't find record in 'orders', Error_code: 1032
```

Fix by inserting the missing row with the correct data from the source:

```sql
-- On the source, find the row
SELECT * FROM mydb.orders WHERE id = 99;

-- On the replica, insert it
INSERT INTO mydb.orders VALUES (99, 'pending', 49.99, 1234, NOW());

-- Restart replica
START REPLICA SQL_THREAD;
```

### Error 1050: Table Already Exists

```text
Last_SQL_Error: Error executing query. Could not execute Query event;
Table 'orders_temp' already exists, Error_code: 1050
```

```sql
STOP REPLICA SQL_THREAD;
DROP TABLE IF EXISTS mydb.orders_temp;
START REPLICA SQL_THREAD;
```

## Skipping a Single Error (Emergency Use Only)

If data integrity allows it, you can skip one event:

```sql
STOP REPLICA SQL_THREAD;
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA SQL_THREAD;
SHOW REPLICA STATUS\G
```

For GTID-based replication, inject an empty transaction instead:

```sql
STOP REPLICA SQL_THREAD;
-- Get the failing GTID from Last_SQL_Error or SHOW REPLICA STATUS
SET GTID_NEXT = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:12345';
BEGIN;
COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA SQL_THREAD;
```

## Configuring Error Filters (Permanent Skip)

For specific errors you want to always skip (use with extreme caution):

```ini
[mysqld]
# Skip specific error codes on replica
replica_skip_errors = 1062,1032
```

Only use `replica_skip_errors` when you have a data validation process to catch and reconcile diverged data.

## Checking for Data Divergence

After resolving errors, check if data has drifted using `pt-table-checksum`:

```bash
pt-table-checksum \
  --host=source.db.example.com \
  --user=root \
  --password=pass \
  --replicate=percona.checksums \
  --databases=mydb
```

Then check results on the replica:

```bash
pt-table-sync \
  --replicate=percona.checksums \
  h=source.db.example.com,D=mydb,t=orders \
  h=replica.db.example.com \
  --print
```

## Automated Monitoring and Alerting

Monitor for replication errors with a shell script:

```bash
#!/bin/bash
STATUS=$(mysql -u root -p"$MYSQL_PASS" -e "SHOW REPLICA STATUS\G" 2>/dev/null)

SQL_RUNNING=$(echo "$STATUS" | grep "Replica_SQL_Running:" | awk '{print $2}')
LAST_ERROR=$(echo "$STATUS" | grep "Last_SQL_Error:" | cut -d: -f2-)

if [ "$SQL_RUNNING" != "Yes" ]; then
  echo "ALERT: Replica SQL thread stopped"
  echo "Error: $LAST_ERROR"
  # Send notification (email, Slack, PagerDuty, etc.)
fi
```

## Summary

Replication errors stop the SQL thread and require manual intervention. Always check `SHOW REPLICA STATUS\G` first to identify the error code and message. Common fixes include deleting duplicate rows (1062), inserting missing rows (1032), or dropping already-existing objects (1050). Use `sql_replica_skip_counter` or GTID empty transactions to skip errors only as a last resort. After any fix, verify data consistency with `pt-table-checksum` and set up monitoring to catch future errors immediately.
