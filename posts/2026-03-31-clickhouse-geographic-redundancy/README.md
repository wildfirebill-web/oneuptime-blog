# How to Set Up Geographic Redundancy for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geographic Redundancy, Disaster Recovery, Replication, Multi-Region

Description: Learn how to configure geographically redundant ClickHouse deployments across multiple data centers or cloud regions for disaster recovery.

---

## Why Geographic Redundancy?

A single data center or region failure can take down your entire ClickHouse cluster. Geographic redundancy places replicas in separate physical locations so that a regional outage does not cause data loss or extended downtime.

## Architecture Options

```text
Option A: Cross-datacenter replication (same cluster)
  - Replicas in DC1 and DC2 share ZooKeeper quorum
  - Writes go to one DC, replicate to the other
  - Simple but ZooKeeper latency affects write speed

Option B: Remote table engine (async cross-region copy)
  - Independent clusters in each region
  - Data copied using RemoteSecure() or the remote() function
  - Eventual consistency; no cross-region ZooKeeper dependency
```

## Option A: Cross-Datacenter ReplicatedMergeTree

Configure the cluster to span two data centers:

```xml
<remote_servers>
  <geo_cluster>
    <shard>
      <replica>
        <host>ch-dc1-node1</host>
        <port>9000</port>
      </replica>
      <replica>
        <host>ch-dc2-node1</host>
        <port>9000</port>
      </replica>
    </shard>
  </geo_cluster>
</remote_servers>
```

Place ZooKeeper nodes in DC1, DC2, and a tie-breaker in a third location to avoid split-brain.

## Option B: Async Replication with Remote()

Set up a materialized view or scheduled job to copy data to the remote region:

```sql
-- Run on DC2 periodically to pull new data from DC1
INSERT INTO events_local
SELECT * FROM remote('ch-dc1-node1', currentDatabase(), 'events_local', 'replica_user', 'password')
WHERE toDate(created_at) = today() AND id NOT IN (SELECT id FROM events_local WHERE toDate(created_at) = today());
```

For large tables, use incremental copying based on a watermark column.

## Using RemoteSecure for Encrypted Cross-Region Traffic

```sql
SELECT count()
FROM remoteSecure('ch-dc2-node1:9440', 'analytics', 'events_local', 'user', 'password')
WHERE toDate(created_at) = today();
```

## Configuring TLS for Cross-Region Connections

```xml
<openSSL>
  <client>
    <caConfig>/etc/ssl/certs/ca-bundle.crt</caConfig>
    <verificationMode>relaxed</verificationMode>
  </client>
</openSSL>
```

## Setting Data Center Preference for Reads

Prefer the local DC replica for reads to reduce latency:

```sql
SET prefer_localhost_replica = 1;
SET load_balancing = 'nearest_hostname';
```

## Monitoring Cross-Region Replication Lag

```sql
SELECT
    replica_name,
    log_max_index - log_pointer AS lag_entries,
    queue_size
FROM system.replicas
WHERE table = 'events_local'
ORDER BY lag_entries DESC;
```

High lag in the remote DC is normal due to WAN latency. Alert when lag grows continuously rather than on absolute values.

## Summary

Geographic redundancy for ClickHouse can be achieved by extending `ReplicatedMergeTree` across data centers (tight coupling) or by setting up independent clusters with async copying via `remote()` or `remoteSecure()` (loose coupling). Choose based on your RPO requirements. Tight coupling gives near-zero RPO but adds ZooKeeper WAN latency to writes; async copying is simpler but allows a window of data loss.
