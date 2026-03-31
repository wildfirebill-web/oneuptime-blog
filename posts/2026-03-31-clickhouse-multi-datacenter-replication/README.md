# How to Set Up Multi-Datacenter ClickHouse Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Multi-Datacenter, Disaster Recovery, ReplicatedMergeTree, Cluster

Description: Set up ClickHouse replication across multiple datacenters for disaster recovery and geographic redundancy using ReplicatedMergeTree and async replication.

---

Multi-datacenter ClickHouse replication allows your cluster to survive a full datacenter outage. The key design decisions are replication topology, Keeper placement, and write consistency vs. latency tradeoffs.

## Topology Options

**Active-Passive**: One datacenter handles all writes; the second is a hot standby. Simpler to operate, some write latency savings.

**Active-Active**: Both datacenters accept writes. Requires careful conflict handling and higher Keeper quorum requirements.

This guide covers Active-Passive with async cross-DC replication.

## Keeper Placement

For cross-DC quorum, spread Keeper nodes across datacenters. A 3-node Keeper with 2 nodes in DC1 and 1 in DC2 tolerates one DC1 node failure but not DC2 failure. For true DC-level fault tolerance, use 5 nodes (3 in DC1, 2 in DC2 - or 3+3 with a tiebreaker):

```xml
<raft_configuration>
  <server><id>1</id><hostname>dc1-ch1</hostname><port>9234</port></server>
  <server><id>2</id><hostname>dc1-ch2</hostname><port>9234</port></server>
  <server><id>3</id><hostname>dc1-ch3</hostname><port>9234</port></server>
  <server><id>4</id><hostname>dc2-ch1</hostname><port>9234</port></server>
  <server><id>5</id><hostname>dc2-ch2</hostname><port>9234</port></server>
</raft_configuration>
```

## Cluster Configuration

```xml
<remote_servers>
  <multi_dc_cluster>
    <shard>
      <replica><host>dc1-ch1</host><port>9000</port></replica>
      <replica><host>dc1-ch2</host><port>9000</port></replica>
      <replica><host>dc2-ch1</host><port>9000</port></replica>
      <replica><host>dc2-ch2</host><port>9000</port></replica>
    </shard>
  </multi_dc_cluster>
</remote_servers>
```

## Create a Replicated Table

```sql
CREATE TABLE events ON CLUSTER multi_dc_cluster (
    event_time DateTime,
    user_id    UInt64,
    payload    String
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

## Async Replication Lag Monitoring

Cross-DC replication is asynchronous. Monitor lag to detect issues before they become critical:

```sql
SELECT replica_name, last_queue_update, inserts_in_queue, queue_size
FROM system.replicas
WHERE table = 'events'
ORDER BY replica_name;
```

Alert when `queue_size > 10000` or `last_queue_update` is more than 5 minutes old.

## Network Considerations

- Open TCP 9000 (native protocol) and 9009 (inter-server replication) between DCs
- Use compressed traffic: `<compression>1</compression>` in replica config
- Set `max_network_bandwidth_for_replication` to prevent replication from saturating the DC link

```xml
<max_network_bandwidth_for_replication>104857600</max_network_bandwidth_for_replication>
```

## Failover Procedure

If DC1 goes down, promote DC2 replicas to accept writes:

```bash
# On DC2 node, verify replica is up to date
clickhouse-client --query "SELECT * FROM system.replicas WHERE is_readonly = 1"

# If readonly due to Keeper quorum loss, use allow_deprecated_error_prone_window_functions
# or reconfigure Keeper with DC2 nodes only
```

## Summary

Multi-datacenter ClickHouse replication requires careful Keeper placement for quorum survivability, bandwidth controls to protect DC links, and monitoring of replication lag to detect divergence early. For active-passive setups, pre-test your failover procedure so that DC2 can accept writes quickly when needed.
