# How to Debug Replication Issues in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Debugging, Troubleshooting, High Availability

Description: Learn how to debug ClickHouse replication issues including queue backlogs, diverged replicas, and ZooKeeper connectivity problems.

---

## Monitoring Replication Health

The primary view for replication status:

```sql
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    total_replicas,
    active_replicas,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    log_max_index - log_pointer AS replication_lag
FROM system.replicas
ORDER BY replication_lag DESC;
```

## Common Replication Problems

### Problem 1: Growing Replication Queue

```sql
SELECT
    database,
    table,
    queue_size,
    inserts_in_queue,
    log_max_index - log_pointer AS lag
FROM system.replicas
WHERE queue_size > 100;
```

**Cause:** Replica cannot fetch data parts fast enough.

**Fix:** Check disk I/O on the lagging node, verify network between nodes, and check for disk space issues.

### Problem 2: Replica in Read-Only Mode

```sql
SELECT database, table, is_readonly, is_session_expired
FROM system.replicas
WHERE is_readonly = 1;
```

**Cause:** ZooKeeper session expired or lost.

**Fix:**

```bash
sudo systemctl restart clickhouse-server
```

Or manually re-activate:

```sql
SYSTEM RESTART REPLICA events_local;
```

### Problem 3: ZooKeeper Connectivity

```sql
SELECT * FROM system.zookeeper
WHERE path = '/clickhouse/tables/default/events';
```

If this query hangs or returns an error, ZooKeeper is unreachable.

```bash
# Test ZooKeeper connectivity
echo "stat" | nc zookeeper-host 2181
```

## Replication Queue Details

```sql
SELECT
    type,
    new_part_name,
    num_tries,
    last_exception,
    last_attempt_time
FROM system.replication_queue
WHERE database = 'default'
  AND table = 'events_local'
ORDER BY last_attempt_time DESC
LIMIT 20;
```

Look for entries with high `num_tries` and read `last_exception` to find the root cause.

## Identifying Diverged Parts

```sql
SELECT
    replica_path,
    replica_name,
    log_max_index,
    log_pointer
FROM system.replicas
WHERE table = 'events_local';
```

If `log_pointer` values are far apart across replicas, they have diverged.

## Force Re-Sync

To make a lagging replica catch up immediately:

```sql
SYSTEM SYNC REPLICA events_local;
```

This blocks until the replica applies all pending queue entries.

## Recovering a Completely Diverged Replica

```bash
# 1. Stop ClickHouse on the bad replica
sudo systemctl stop clickhouse-server

# 2. Remove the ZooKeeper entry for this replica
clickhouse-client --host ch-node-1 --query "
  SYSTEM DROP REPLICA 'replica-2' FROM TABLE default.events_local
"

# 3. Restart - ClickHouse will re-initialize from the healthy replica
sudo systemctl start clickhouse-server
```

## Monitoring Replication Lag in Prometheus

```sql
SELECT
    concat(database, '.', table) AS table_name,
    log_max_index - log_pointer AS lag
FROM system.replicas;
```

Alert when lag consistently exceeds a threshold (for example 1000 entries).

## Summary

Debug ClickHouse replication issues using `system.replicas` for health status, `system.replication_queue` for detailed queue entries with error messages, and `system.zookeeper` for Keeper connectivity. For lagging replicas, use `SYSTEM SYNC REPLICA` to force catch-up. For completely diverged replicas, use `SYSTEM DROP REPLICA` and restart the node to re-sync from scratch.
