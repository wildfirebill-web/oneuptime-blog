# How to Respond to MySQL Failover Events

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Failover, Replication, High Availability, Incident Response

Description: A structured guide for responding to MySQL failover events, validating replica promotion, and restoring service with minimal data loss.

---

## Understanding MySQL Failover

A MySQL failover event occurs when the primary database server becomes unavailable and traffic must be redirected to a replica. Failover can be automatic (using tools like MHA, Orchestrator, or ProxySQL) or manual. Either way, the on-call engineer must verify the failover completed cleanly and validate data integrity.

## Detecting a Failover

Monitoring alerts typically fire when the primary stops responding. Check the current replication state immediately:

```sql
-- Run on the promoted replica (new primary)
SHOW MASTER STATUS\G

-- Verify replication is no longer running on the new primary
SHOW SLAVE STATUS\G
```

Check the application configuration or load balancer to confirm traffic is routing to the new primary.

## Verifying the Promoted Replica

Before accepting write traffic, validate the promoted replica:

```sql
-- Check the replica has caught up to the old primary's binlog position
SHOW MASTER STATUS\G

-- Verify read_only is disabled on the new primary
SHOW GLOBAL VARIABLES LIKE 'read_only';
SHOW GLOBAL VARIABLES LIKE 'super_read_only';
```

If `read_only` is still enabled, disable it:

```sql
SET GLOBAL read_only = 0;
SET GLOBAL super_read_only = 0;
```

## Validating Data Integrity

Check for any replication lag that may have caused data loss at the time of failover:

```bash
# If using GTID-based replication, check for missing transactions
mysql -u root -p -e "
  SELECT @@global.gtid_executed AS executed_gtids,
         @@global.gtid_purged AS purged_gtids;
"
```

Run a quick sanity check on critical tables:

```sql
-- Compare row counts on promoted replica vs expected values
SELECT table_name, table_rows
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
ORDER BY table_rows DESC
LIMIT 20;
```

## Updating Application Configuration

Update your application to point to the new primary:

```bash
# If using a config file
sed -i 's/old-primary.db.example.com/new-primary.db.example.com/g' /etc/app/database.conf

# Reload the application
sudo systemctl reload myapp
```

If using ProxySQL, the failover may already be handled automatically. Verify:

```bash
mysql -u admin -p -h 127.0.0.1 -P 6032 -e "SELECT hostgroup_id, hostname, status FROM mysql_servers;"
```

## Rebuilding the Failed Primary as a Replica

Once the incident is resolved, rebuild the failed server as a replica of the new primary:

```sql
-- On the old primary (now being rebuilt as a replica)
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST = 'new-primary.db.example.com',
  MASTER_USER = 'replication_user',
  MASTER_PASSWORD = 'repl_password',
  MASTER_AUTO_POSITION = 1;

START SLAVE;
SHOW SLAVE STATUS\G
```

Monitor `Seconds_Behind_Master` until it reaches 0.

## Post-Failover Checklist

```text
[ ] Verify new primary is accepting writes
[ ] Confirm application connections are routing correctly
[ ] Check for any replication errors on remaining replicas
[ ] Validate data integrity with row count checks
[ ] Review error logs on the failed primary for root cause
[ ] Update monitoring dashboards to reflect new topology
[ ] Document the timeline and actions taken
[ ] Schedule a post-mortem within 48 hours
```

## Summary

Responding to MySQL failover events requires quickly verifying the promoted replica's status, disabling `read_only` mode, validating data integrity, and updating application configuration to route writes correctly. After stabilizing the incident, the failed primary should be rebuilt as a new replica. A thorough post-mortem helps prevent recurrence and improves failover automation.
