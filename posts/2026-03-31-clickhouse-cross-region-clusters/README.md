# How to Set Up Cross-Region ClickHouse Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cross-Region, Replication, Cluster, Disaster Recovery, Network

Description: Learn how to set up ClickHouse clusters spanning multiple cloud regions, including network configuration, replication tuning, and latency management.

---

Running ClickHouse across multiple cloud regions provides disaster recovery and geographic redundancy, but introduces new challenges around latency, bandwidth, and consistency. This guide covers the key steps and tradeoffs.

## Architecture Considerations

Cross-region clusters typically use one of these patterns:

- **Replicated cluster**: same shard replicated across regions (read-local, write-anywhere)
- **Multi-shard multi-region**: different shards in different regions (data partitioned by region)

The replicated approach is simpler and provides full redundancy. The sharded approach reduces cross-region traffic but complicates queries.

## Network Requirements

Open these ports between regions:

```text
9000/tcp  - native client protocol
9009/tcp  - inter-server replication
9181/tcp  - ClickHouse Keeper
9234/tcp  - Keeper Raft consensus
```

Use private networking (VPC peering, VPN, or dedicated interconnect) rather than the public internet to reduce latency and cost.

## Bandwidth Throttling

Limit replication bandwidth to protect your cross-region link:

```xml
<max_network_bandwidth_for_replication>52428800</max_network_bandwidth_for_replication>
<!-- 50 MB/s per replica connection -->
```

## Keeper Placement for Cross-Region Quorum

A 5-node Keeper ensemble with 3 nodes in region A and 2 nodes in region B survives region A partial failure (1 node down) but not full region B loss. For true region-level fault tolerance, use 3 nodes in region A, 3 in region B, and accept that Raft quorum (4 of 6) requires both regions to be up.

```xml
<raft_configuration>
  <server><id>1</id><hostname>us-east-ch1</hostname><port>9234</port></server>
  <server><id>2</id><hostname>us-east-ch2</hostname><port>9234</port></server>
  <server><id>3</id><hostname>us-east-ch3</hostname><port>9234</port></server>
  <server><id>4</id><hostname>eu-west-ch1</hostname><port>9234</port></server>
  <server><id>5</id><hostname>eu-west-ch2</hostname><port>9234</port></server>
</raft_configuration>
```

## Configuring Replication Timeouts for High Latency

Cross-region latency (50-150ms RTT) requires looser replication timeouts:

```xml
<coordination_settings>
  <operation_timeout_ms>30000</operation_timeout_ms>
  <session_timeout_ms>60000</session_timeout_ms>
  <heart_beat_interval_ms>2000</heart_beat_interval_ms>
  <election_timeout_lower_bound_ms>10000</election_timeout_lower_bound_ms>
  <election_timeout_upper_bound_ms>20000</election_timeout_upper_bound_ms>
</coordination_settings>
```

## Read-Local Queries

Configure hedged requests so cross-region queries prefer local replicas:

```sql
SET use_hedged_requests = 1;
SET max_parallel_replicas = 2;
```

Or use a dedicated replica group that lists local nodes first in the cluster config.

## Monitor Cross-Region Replication Lag

```sql
SELECT replica_name, queue_size, last_queue_update,
       dateDiff('second', last_queue_update, now()) AS lag_seconds
FROM system.replicas
WHERE table = 'events'
ORDER BY lag_seconds DESC;
```

Alert when `lag_seconds > 300` (5 minutes) to catch network issues early.

## Summary

Cross-region ClickHouse clusters require careful Keeper placement for quorum survivability, bandwidth throttling to protect inter-region links, and relaxed timeout settings to handle higher RTT. Monitor replication lag continuously and test your failover procedure before relying on it in production.
