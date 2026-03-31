# How to Choose Sharding Keys for Distributed Tables in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Sharding, Distributed Table, Sharding Key, Query Optimization, Cluster

Description: Learn how to choose effective sharding keys for ClickHouse Distributed tables to ensure even data distribution and efficient query routing.

---

The sharding key is one of the most important decisions when setting up a distributed ClickHouse cluster. A poor choice leads to hot shards, slow cross-shard queries, and uneven disk usage.

## Goals of a Good Sharding Key

1. **Even distribution**: rows spread uniformly across shards
2. **Query colocation**: rows queried together land on the same shard to avoid cross-shard joins
3. **Low cardinality cost**: computing the key at insert time should be cheap

## Avoid Low-Cardinality Keys

Using a column with few distinct values as the sharding key creates hot shards:

```sql
-- Bad: only ~10 statuses, most shards will be empty
ENGINE = Distributed('cluster', 'db', 'events_local', status)
```

## Use a Hash of a High-Cardinality Column

```sql
-- Good: hashing user_id gives even distribution
ENGINE = Distributed('cluster', 'db', 'events_local', intHash64(user_id))
```

`intHash64` is fast and produces a uniform distribution. For UUIDs, use `cityHash64(uuid_col)`.

## Choose the Key Based on Query Patterns

If most queries filter by `tenant_id`, shard by tenant so each tenant's data lives on one shard. This avoids cross-shard scatter-gather for tenant-scoped queries:

```sql
ENGINE = Distributed('cluster', 'db', 'events_local', intHash64(tenant_id))
```

All events for `tenant_id = 42` land on the same shard. A query with `WHERE tenant_id = 42` only touches one shard.

## Composite Sharding Keys

Combine multiple columns for compound distribution:

```sql
ENGINE = Distributed('cluster', 'db', 'events_local',
    cityHash64(tenant_id, user_id))
```

This distributes at the `(tenant, user)` grain, which colocates per-user data while avoiding tenant-level hot spots for tenants with many users.

## Verify Distribution Evenness

After loading data, check row counts per shard:

```sql
SELECT _shard_num, count() AS rows
FROM events
GROUP BY _shard_num
ORDER BY _shard_num;
```

Ideally, shard counts are within 10-15% of each other.

## Use rand() for Purely Even Distribution

If queries always scatter to all shards and you only care about even disk use:

```sql
ENGINE = Distributed('cluster', 'db', 'events_local', rand())
```

This gives the best distribution but loses any colocation benefits.

## Avoid Changing the Sharding Key

Once data is loaded, changing the sharding key requires re-inserting all data. Plan carefully upfront, and consider using a synthetic key derived from a stable business identifier.

## Summary

Choose a sharding key that is high-cardinality, maps to your primary query filter, and is cheap to compute. Hash functions like `intHash64` and `cityHash64` provide even distribution. Verify distribution after initial load and monitor shard sizes over time to catch imbalances early.
