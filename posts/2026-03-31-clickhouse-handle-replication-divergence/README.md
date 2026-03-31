# How to Handle ClickHouse Replication Divergence

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Data Consistency, Incident Response, Operations

Description: Detect and resolve ClickHouse replication divergence where replicas have different data, using checksum comparison and controlled part re-fetch procedures.

---

Replication divergence happens when replicas have inconsistent data - different parts, different checksums, or different row counts. This is a serious condition that must be fixed before it causes incorrect query results.

## Detect Divergence

Compare row counts across replicas:

```sql
-- Run on each replica separately and compare
SELECT count() FROM events WHERE ts >= today() - 7;
```

Check for part checksum mismatches:

```sql
SELECT
    name,
    rows,
    checksums_sha512,
    active
FROM system.parts
WHERE table = 'events' AND active
ORDER BY name;
```

Compare the output from each replica. Mismatched `checksums_sha512` values indicate divergence.

## Identify the Root Cause

Check the replication error log:

```sql
SELECT *
FROM system.replication_queue
WHERE last_exception != ''
ORDER BY last_attempt_time DESC
LIMIT 20;
```

Common causes:
- Network partition that caused a split-brain write
- Manual DDL run on only one replica
- Disk corruption on one replica

## Resolving Part-Level Divergence

For a single bad part, detach it on the diverged replica and allow re-fetch:

```bash
# Identify the bad part name from the comparison above
PART_NAME="20240101_1_1_0"
TABLE="events"
```

```sql
-- On the diverged replica
ALTER TABLE events DETACH PART '20240101_1_1_0';

-- The ReplicatedMergeTree will re-fetch from the healthy replica
```

Monitor the queue:

```sql
SELECT * FROM system.replication_queue WHERE table = 'events';
```

## Full Replica Rebuild

If divergence is widespread, rebuild the replica from scratch:

```sql
-- On the diverged replica, drop and recreate
DROP TABLE events;

-- Recreate with the same ReplicatedMergeTree path
CREATE TABLE events ... ENGINE = ReplicatedMergeTree('/clickhouse/tables/shard1/events', 'replica2') ...;
```

The table refetches all parts from healthy replicas. Monitor `absolute_delay` in `system.replicas` until it reaches zero.

## Monitoring

Configure [OneUptime](https://oneuptime.com) to alert when `system.replication_queue` contains entries with non-empty `last_exception` for more than 10 minutes. This catches divergence early, before it becomes widespread.

## Summary

Replication divergence in ClickHouse is fixed by detaching bad parts or rebuilding the replica. Always identify the root cause to prevent recurrence, and monitor the replication queue for errors daily.
