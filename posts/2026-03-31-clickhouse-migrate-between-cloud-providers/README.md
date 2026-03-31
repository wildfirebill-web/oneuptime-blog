# How to Migrate ClickHouse Between Cloud Providers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Migration, Cloud, Backup, Data Transfer

Description: Migrate a ClickHouse cluster from one cloud provider to another with minimal downtime using clickhouse-backup, remote table functions, and replication.

---

Migrating ClickHouse between clouds requires careful planning to avoid data loss and minimize downtime. The best approach depends on data volume: small datasets suit direct copy; large ones need replication-based migration.

## Option 1 - Backup and Restore (Small Datasets)

```bash
# On source cluster
clickhouse-backup create pre_migration_$(date +%Y%m%d)
clickhouse-backup upload pre_migration_$(date +%Y%m%d)

# On destination cluster (new cloud)
clickhouse-backup download pre_migration_$(date +%Y%m%d)
clickhouse-backup restore pre_migration_$(date +%Y%m%d)
```

Verify row counts after restore:

```sql
-- Source
SELECT table, sum(rows) FROM system.parts WHERE active GROUP BY table;

-- Destination (must match)
SELECT table, sum(rows) FROM system.parts WHERE active GROUP BY table;
```

## Option 2 - Remote Table Function (Medium Datasets)

Copy data directly between clusters over the network:

```sql
-- Run on destination cluster
INSERT INTO events_new
SELECT *
FROM remote('source-clickhouse:9000', mydb.events, 'user', 'password')
WHERE ts >= '2024-01-01';
```

## Option 3 - Replication-Based Migration (Large Datasets)

Add the destination cluster as a replica of the source:

1. Add destination nodes to the same ReplicatedMergeTree path in Keeper.
2. Let replication catch up (monitor `system.replicas`).
3. Redirect writes to the destination nodes.
4. Remove source nodes from the replica set.

```sql
-- Check replication progress on destination
SELECT
    replica_path,
    absolute_delay,
    queue_size
FROM system.replicas
WHERE table = 'events';
```

## Cutover Steps

```text
1. Stop writes on source (maintenance window or feature flag).
2. Wait for replication lag to reach zero.
3. Update application connection strings to destination.
4. Verify a test query returns expected results.
5. Decommission source nodes after 48-hour observation period.
```

## Monitoring During Migration

Use [OneUptime](https://oneuptime.com) to track write error rates and query latency on both clusters during the migration window. Create a status page entry so stakeholders can follow progress.

## Summary

Small migrations use backup/restore; large ones use replication. The replication approach provides near-zero downtime by letting the destination catch up before you cut over application traffic.
