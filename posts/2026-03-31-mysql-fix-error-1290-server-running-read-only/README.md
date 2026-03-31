# How to Fix ERROR 1290 MySQL Server Running with --read-only Option

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Read-Only, Error, Replication, Configuration

Description: Fix MySQL ERROR 1290 by disabling the read_only or super_read_only variable, or redirecting writes to the correct primary node in your replication setup.

---

MySQL ERROR 1290 appears when you try to write to a server that has been set to read-only mode. The message reads: `ERROR 1290 (HY000): The MySQL server is running with the --read-only option so it cannot execute this statement`. This is common on replica servers that should only accept reads.

## Check the Current Read-Only Status

```sql
SHOW VARIABLES LIKE 'read_only';
SHOW VARIABLES LIKE 'super_read_only';
SHOW VARIABLES LIKE 'transaction_read_only';
```

There are two levels: `read_only` restricts non-SUPER users, and `super_read_only` restricts even SUPER users (used on replicas to prevent accidental writes).

## Why This Mode Is Active

Common reasons:
- The server is a replica in a replication setup
- The DBA manually enabled it for maintenance
- A cloud provider (RDS, Cloud SQL) enables it on read replicas automatically
- The `--read-only` flag is set in `my.cnf` or the startup command

## Fix: Disable read_only Temporarily

If you intentionally need to write to this server (for example, during a failover or emergency fix):

```sql
-- Requires SUPER or SYSTEM_VARIABLES_ADMIN privilege
SET GLOBAL read_only = OFF;

-- If super_read_only is also set, disable it first
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
```

This change is not persistent. After a MySQL restart, the value from `my.cnf` or the startup flags will be used again.

## Fix: Update my.cnf for a Permanent Change

To make the server permanently writable, remove or change the setting in `my.cnf`:

```text
[mysqld]
# Remove or comment out this line
# read_only = ON

# Or explicitly set to OFF
read_only = OFF
```

Then restart MySQL:

```bash
sudo systemctl restart mysql
```

## Fix: Redirect Writes to the Primary

The correct fix for a replica server is to send writes to the primary instead:

```sql
-- On the replica, find the primary host
SHOW REPLICA STATUS\G
-- Look at Source_Host

-- On the primary, verify it is writable
SHOW VARIABLES LIKE 'read_only';
```

In your application, use separate connection strings for reads (replica) and writes (primary):

```python
DATABASES = {
    'default': {'HOST': 'mysql-primary'},
    'replica': {'HOST': 'mysql-replica', 'TEST': {'MIRROR': 'default'}},
}
DATABASE_ROUTERS = ['myapp.routers.ReadReplicaRouter']
```

## Fix: Cloud Provider Read Replicas

On AWS RDS or Google Cloud SQL read replicas, `read_only` is automatically managed. To write, promote the replica to a standalone instance or use the primary endpoint.

```bash
# AWS RDS - promote read replica to standalone
aws rds promote-read-replica --db-instance-identifier my-replica
```

## Verify After Fix

```sql
SHOW VARIABLES LIKE 'read_only';
SHOW VARIABLES LIKE 'super_read_only';

-- Test with a write
CREATE TABLE test_write_check (id INT PRIMARY KEY);
DROP TABLE test_write_check;
```

## Summary

ERROR 1290 means the server intentionally blocks writes. For replicas, redirect writes to the primary rather than disabling read-only mode. For maintenance windows or emergency fixes, use `SET GLOBAL super_read_only = OFF` followed by `SET GLOBAL read_only = OFF`. Always re-enable read-only mode on replicas after maintenance to prevent accidental writes that break replication.
