# How to Handle ClickHouse Cluster Expansion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Scaling, Cluster Expansion, Sharding, Operations

Description: Expand a ClickHouse cluster by adding new shards or replicas without downtime, including config updates, data redistribution, and validation steps.

---

Cluster expansion is required when storage or write throughput approaches capacity limits. ClickHouse supports live expansion through replica addition (no data move) or shard addition (requires data migration).

## Adding a Replica (No Data Migration)

Adding a replica to an existing shard is the simplest expansion. The new node automatically fetches all parts from existing replicas.

1. Provision the new node with the same ClickHouse version.

2. Update `config.xml` on all nodes to include the new replica in the cluster:

```xml
<cluster>
    <shard>
        <replica><host>ch-01</host><port>9000</port></replica>
        <replica><host>ch-02</host><port>9000</port></replica>
        <replica><host>ch-03-new</host><port>9000</port></replica>  <!-- new -->
    </shard>
</cluster>
```

3. On the new node, create all replicated tables with the same Keeper paths but a new replica identifier:

```sql
CREATE TABLE events
ENGINE = ReplicatedMergeTree('/clickhouse/tables/shard1/events', 'replica3')
PARTITION BY toYYYYMM(ts)
ORDER BY (id, ts);
```

4. Monitor catch-up:

```sql
SELECT table, absolute_delay, queue_size
FROM system.replicas
ORDER BY absolute_delay DESC;
```

## Adding a New Shard (With Data Migration)

Adding a shard redistributes future writes but requires migrating historical data.

1. Add the new shard to the cluster config.
2. Create tables on new shard nodes.
3. Migrate existing data using `INSERT INTO ... SELECT`:

```sql
-- Run on new shard, pull data for keys that hash to new shard
INSERT INTO events_local
SELECT *
FROM remote('old-ch-node:9000', default.events, 'user', 'pass')
WHERE cityHash64(user_id) % new_shard_count = new_shard_id;
```

4. Update the distributed table to include the new shard.

## Validating Expansion

```sql
-- Check row counts per shard
SELECT _shard_num, count()
FROM dist_events
GROUP BY _shard_num
ORDER BY _shard_num;
```

Counts should be approximately equal after redistribution.

## Monitoring During Expansion

Track replication queue size and disk usage on new nodes via [OneUptime](https://oneuptime.com). Alert if replication lag on the new replica exceeds 30 minutes - this may indicate a network or disk bottleneck slowing the initial sync.

## Summary

Adding replicas requires no data migration and is the safest form of expansion. Adding shards increases write throughput and storage capacity but requires migrating historical data. Always validate row counts across shards after expansion.
