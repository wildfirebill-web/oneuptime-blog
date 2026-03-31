# How to Replicate ClickHouse Data Across Cloud Regions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Multi-Region, High Availability, Disaster Recovery

Description: Set up ClickHouse cross-region replication using ReplicatedMergeTree and ClickHouse Keeper to keep data consistent across geographically distributed nodes.

---

Cross-region replication protects against a full region outage and enables local reads for globally distributed users. ClickHouse achieves this through ReplicatedMergeTree, which uses Keeper for coordination metadata.

## Prerequisites

- Two or more cloud regions with VPN or private interconnect.
- ClickHouse Keeper running as a 3-node (or 5-node) ensemble spanning regions.
- Consistent ClickHouse versions across all nodes.

## Keeper Placement

For a two-region setup (us-east-1 and eu-west-1), place two Keeper nodes in the primary region and one in the secondary. This keeps quorum available during a secondary region failure.

```xml
<raft_configuration>
    <server><id>1</id><hostname>keeper-us-1</hostname><port>9234</port></server>
    <server><id>2</id><hostname>keeper-us-2</hostname><port>9234</port></server>
    <server><id>3</id><hostname>keeper-eu-1</hostname><port>9234</port></server>
</raft_configuration>
```

## Remote Disk on S3

Use S3 with cross-region replication enabled at the bucket level as the storage backend. Both regions read from buckets replicated by AWS S3 CRR:

```xml
<disks>
    <s3_us><type>s3</type><endpoint>https://s3.us-east-1.amazonaws.com/ch-us/</endpoint></s3_us>
    <s3_eu><type>s3</type><endpoint>https://s3.eu-west-1.amazonaws.com/ch-eu/</endpoint></s3_eu>
</disks>
```

## Replicated Table

```sql
CREATE TABLE logs ON CLUSTER 'cross_region'
(
    id      UInt64,
    message String,
    ts      DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/logs',
    '{replica}'
)
PARTITION BY toYYYYMM(ts)
ORDER BY (id, ts);
```

## Verifying Replication Health

```sql
SELECT
    replica_path,
    is_leader,
    inserts_in_queue,
    queue_size,
    last_queue_update
FROM system.replicas
WHERE table = 'logs';
```

## Failover Procedure

If the primary region fails, promote a replica in the secondary region by updating the load balancer target. Keeper retains the replication metadata, so the replica can become leader once the network partition resolves.

## Monitoring

Track `queue_size` and `last_queue_update` via [OneUptime](https://oneuptime.com). Alert when either replica has more than 1,000 queued operations or has not updated in 2 minutes.

## Summary

Cross-region ClickHouse replication relies on ReplicatedMergeTree and Keeper. Careful Keeper placement ensures quorum survives a single-region failure, while S3 cross-region replication provides durable, geographically redundant storage.
