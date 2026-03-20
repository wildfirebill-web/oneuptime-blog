# How to Troubleshoot Rancher Database Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Database, MySQL

Description: Diagnose and resolve Rancher database connection failures, including MySQL connectivity issues, connection pool exhaustion, and credential problems.

## Introduction

Rancher HA installations rely on an external MySQL (or MySQL-compatible) database to store cluster state, settings, and RBAC data. Database connectivity issues cause Rancher to fail to start, lose cluster state, or become unresponsive. This guide covers how to identify and resolve the most common database issues.

## Understanding Rancher's Database Usage

- **Single-node Rancher (K3s)**: Uses embedded SQLite or an external datastore.
- **Rancher HA (3+ replicas)**: Requires an external MySQL 8.0+ or compatible (e.g., Aurora MySQL, MariaDB 10.6+).
- Rancher connects via the `CATTLE_DB_*` environment variables.

## Step 1: Identify Database Errors in Rancher Logs

```bash
# Check Rancher logs for database-related messages
kubectl logs -n cattle-system -l app=rancher --tail=300 \
  | grep -iE "database|sql|mysql|connection|driver"

# Common error messages:
# "dial tcp: connect: connection refused"     → MySQL not reachable
# "Access denied for user 'rancher'@'x.x.x.x'" → Wrong credentials
# "too many connections"                      → Connection pool exhausted
# "deadlock found when trying to get lock"   → MySQL deadlock
# "Table doesn't exist"                      → Schema migration failed
```

## Step 2: Verify Database Connection Settings

```bash
# Inspect the database connection environment variables
kubectl get deployment -n cattle-system rancher -o json \
  | jq '.spec.template.spec.containers[].env[] | select(.name | startswith("CATTLE_DB"))'

# Expected variables:
# CATTLE_DB_TYPE       = mysql
# CATTLE_DB_HOST       = db.example.com
# CATTLE_DB_PORT       = 3306
# CATTLE_DB_NAME       = cattle
# CATTLE_DB_USER       = cattle
# CATTLE_DB_PASS       = <from secret>

# Verify the secret contains valid credentials
kubectl get secret -n cattle-system rancher-db-secret -o json \
  | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

## Step 3: Test MySQL Connectivity

```bash
# Test connectivity from inside the cluster
kubectl run mysql-test --rm -it \
  --image=mysql:8 \
  --restart=Never \
  -- mysql \
  -h <db-host> \
  -P 3306 \
  -u cattle \
  -p<password> \
  cattle \
  -e "SELECT 1; SHOW TABLES;"

# Test network reachability without MySQL client
kubectl run tcp-test --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- nc -zv <db-host> 3306
```

## Step 4: Check MySQL Health

Log in directly to the MySQL server:

```sql
-- Check connection count
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- Check for blocked queries
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G

-- Check for long-running queries
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE time > 30
ORDER BY time DESC;

-- Check database size
SELECT table_schema AS database_name,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'cattle'
GROUP BY table_schema;
```

## Step 5: Fix Connection Pool Exhaustion

If `too many connections` is the error:

```bash
# Increase max_connections in MySQL
# In /etc/mysql/mysql.conf.d/mysqld.cnf:
# max_connections = 500

# Restart MySQL to apply
sudo systemctl restart mysql

# On the Rancher side, reduce the connection pool size if there are many replicas
# Each Rancher replica uses ~20-30 connections
# Formula: max_connections >= (rancher_replicas * 30) + overhead

# Update Rancher to limit connection pool
kubectl set env deployment/rancher -n cattle-system \
  CATTLE_DB_MAX_OPEN_CONNECTIONS=25 \
  CATTLE_DB_MAX_IDLE_CONNECTIONS=5
```

## Step 6: Recover from a Failed Schema Migration

```bash
# If Rancher logs show migration errors, check the migration table
mysql -h <db-host> -u cattle -p cattle << 'EOF'
SELECT * FROM schema_migrations ORDER BY version DESC LIMIT 10;
SELECT * FROM cattle_lock WHERE name='data-migration' LIMIT 5;
EOF

# If a migration is stuck (e.g., Rancher crashed mid-migration):
# 1. Scale down Rancher
kubectl scale deployment -n cattle-system rancher --replicas=0

# 2. Remove the lock
mysql -h <db-host> -u cattle -p cattle \
  -e "DELETE FROM cattle_lock WHERE name='data-migration';"

# 3. Scale Rancher back up
kubectl scale deployment -n cattle-system rancher --replicas=3
```

## Step 7: Check Firewall and Security Groups

| Source | Destination | Port | Protocol |
|---|---|---|---|
| Rancher pods CIDR | MySQL server IP | 3306 | TCP |

```bash
# Test from a node (not inside a pod) if the issue is a node-level firewall
telnet <db-host> 3306
# OR
nc -zv <db-host> 3306
```

## Conclusion

Rancher database connection issues typically fall into three categories: network reachability, credential mismatches, or resource exhaustion. Always start with log analysis to identify the exact error, then test connectivity progressively from inside the cluster. Monitoring MySQL `max_connections` and query latency proactively will prevent most production database incidents.
