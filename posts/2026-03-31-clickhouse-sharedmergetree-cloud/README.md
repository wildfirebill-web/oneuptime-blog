# How to Use SharedMergeTree Engine in ClickHouse Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SharedMergeTree, ClickHouse Cloud, Table Engine, Scalability

Description: Learn how SharedMergeTree works in ClickHouse Cloud, how it differs from ReplicatedMergeTree, and why it is the default engine for cloud deployments.

---

SharedMergeTree is the default table engine for ClickHouse Cloud, designed specifically for cloud-native deployments where compute and storage are separated. It replaces ReplicatedMergeTree in cloud environments and offers better scalability, simpler replication management, and lower operational overhead.

## How SharedMergeTree Differs from ReplicatedMergeTree

In self-hosted ClickHouse, ReplicatedMergeTree stores data on local disk and coordinates replicas through ZooKeeper or ClickHouse Keeper. Each replica holds a full copy of the data.

SharedMergeTree uses shared object storage (like S3) as the single source of truth. All compute nodes access the same data in object storage without maintaining independent replicas. This enables:

- Instant horizontal scaling without data rebalancing
- No need for data duplication across nodes
- Separated scaling of compute and storage

## Creating a SharedMergeTree Table

In ClickHouse Cloud, you create tables exactly as you would with MergeTree - the engine is automatically mapped to SharedMergeTree:

```sql
CREATE TABLE events
(
    event_id    UInt64,
    user_id     UInt32,
    event_name  LowCardinality(String),
    event_date  DateTime,
    properties  Map(String, String)
)
ENGINE = MergeTree()
ORDER BY (event_date, user_id)
PARTITION BY toYYYYMM(event_date);
```

In ClickHouse Cloud, this will use SharedMergeTree internally. You can verify with:

```sql
SELECT name, engine FROM system.tables WHERE name = 'events';
-- engine: SharedMergeTree
```

## Writing Data

Inserts work the same way as with any MergeTree engine:

```sql
INSERT INTO events (event_id, user_id, event_name, event_date)
VALUES (1, 1001, 'page_view', now());

-- Bulk insert
INSERT INTO events
SELECT
    rowNumberInAllBlocks() AS event_id,
    user_id,
    event_name,
    event_date
FROM source_table;
```

## Scaling Replicas

One of SharedMergeTree's key advantages is that adding or removing replicas does not require copying data. New nodes simply connect to the shared object storage:

```sql
-- In ClickHouse Cloud, scaling is done via the console or API
-- New nodes pick up from shared storage immediately

-- You can check active replicas
SELECT
    hostname(),
    is_leader,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE table = 'events';
```

## Consistency Model

SharedMergeTree provides a different consistency model than ReplicatedMergeTree. Reads are served from shared storage, so all nodes always see the same data. However, there can be a brief lag between when data is written to object storage and when it is visible to all readers:

```sql
-- Force a sync if needed (usually not required)
SYSTEM SYNC REPLICA events;
```

## Limitations Compared to Self-Hosted MergeTree

- SharedMergeTree is only available in ClickHouse Cloud
- Local disk caching is used for hot data, but cold data reads hit object storage
- Some very low-latency scenarios may perform differently due to object storage access patterns

## TTL and Tiered Storage

SharedMergeTree supports TTL and tiered storage policies just like MergeTree:

```sql
ALTER TABLE events
    MODIFY TTL event_date + INTERVAL 90 DAY;
```

In ClickHouse Cloud, tiering between hot (cached) and cold (direct object storage) happens automatically.

## Summary

SharedMergeTree is the cloud-native evolution of ReplicatedMergeTree, purpose-built for ClickHouse Cloud's shared object storage architecture. It simplifies replica management, enables instant scaling, and eliminates data duplication costs. When working in ClickHouse Cloud, you get SharedMergeTree automatically - just create tables with the standard MergeTree syntax and let the platform handle the rest.
