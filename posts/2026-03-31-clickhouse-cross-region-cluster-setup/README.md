# How to Set Up Cross-Region ClickHouse Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cross-Region, Cluster, Replication, High Availability

Description: Learn how to configure a ClickHouse cluster spanning multiple geographic regions for disaster recovery, data locality, and latency-sensitive workloads.

---

A cross-region ClickHouse cluster enables disaster recovery, read locality, and compliance with data residency requirements. Setting one up requires careful attention to network latency, ZooKeeper quorum placement, and replication configuration.

## Architecture Overview

A typical cross-region setup places shards in each region with replicas spread across regions:

```text
Region US-East:
  Shard 1 - Replica 1 (primary)
  Shard 2 - Replica 1 (primary)

Region EU-West:
  Shard 1 - Replica 2 (cross-region replica)
  Shard 2 - Replica 2 (cross-region replica)
```

Each shard has one replica per region, providing fault tolerance for an entire region failure.

## Cluster Configuration

Define the cross-region cluster in `config.d/clusters.xml` on all nodes:

```xml
<remote_servers>
    <global_cluster>
        <shard>
            <replica>
                <host>ch-us-east-01</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-eu-west-01</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>ch-us-east-02</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-eu-west-02</host>
                <port>9000</port>
            </replica>
        </shard>
    </global_cluster>
</remote_servers>
```

## ZooKeeper Placement for Cross-Region

ZooKeeper quorum must be distributed to survive region failures. Use 5 nodes: 2 in US-East, 2 in EU-West, 1 in a third region as tiebreaker:

```xml
<zookeeper>
    <node><host>zk-us-east-01</host><port>2181</port></node>
    <node><host>zk-us-east-02</host><port>2181</port></node>
    <node><host>zk-eu-west-01</host><port>2181</port></node>
    <node><host>zk-eu-west-02</host><port>2181</port></node>
    <node><host>zk-ap-south-01</host><port>2181</port></node>
</zookeeper>
```

Using ClickHouse Keeper instead of ZooKeeper is preferred for new deployments as it has lower overhead.

## Creating Tables for Cross-Region Replication

```sql
CREATE TABLE events_local ON CLUSTER global_cluster
(
    event_time DateTime,
    region LowCardinality(String),
    user_id UInt64,
    action String
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events_local',
    '{replica}'
)
PARTITION BY toYYYYMM(event_time)
ORDER BY (region, user_id, event_time);
```

## Handling Cross-Region Latency

Cross-region replication adds latency due to WAN round trips. Set appropriate ZooKeeper session timeouts:

```xml
<zookeeper>
    <session_timeout_ms>30000</session_timeout_ms>
    <operation_timeout_ms>10000</operation_timeout_ms>
</zookeeper>
```

For reads, prefer local replicas to minimize latency:

```sql
SET prefer_localhost_replica = 1;
SET load_balancing = 'nearest_hostname';
```

## Monitoring Cross-Region Replication Lag

```sql
SELECT
    replica_name,
    absolute_delay,
    queue_size,
    active_replicas
FROM system.replicas
WHERE absolute_delay > 60
ORDER BY absolute_delay DESC;
```

Cross-region replicas naturally have higher lag than intra-region replicas. Set your alert thresholds accordingly.

## Network Security for Cross-Region

Secure inter-region traffic with TLS:

```xml
<openSSL>
    <server>
        <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
        <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
    </server>
    <client>
        <caConfig>/etc/ssl/certs/ca-bundle.crt</caConfig>
    </client>
</openSSL>
```

Use dedicated VPN tunnels or private network peering between regions rather than routing replication traffic over the public internet.

## Summary

Cross-region ClickHouse clusters require careful placement of ZooKeeper/Keeper nodes for quorum availability, region-aware shard and replica layout, and appropriate timeout settings for WAN latency. Configure `prefer_localhost_replica` and `load_balancing = nearest_hostname` to route reads to local replicas, reducing both latency and cross-region bandwidth costs.
