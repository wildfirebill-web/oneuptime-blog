# How to Handle ClickHouse Keeper Failover

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Keeper, Failover, High Availability, Operation

Description: Learn how ClickHouse Keeper handles leader failover automatically, how to verify your cluster recovers correctly, and what to do when automatic failover does not work.

ClickHouse Keeper is designed to survive node failures automatically. When the leader fails, Raft triggers an election and a follower becomes the new leader within seconds. ClickHouse clients automatically reconnect to the new leader. In most cases you do not need to intervene. But you need to understand the failover process to configure timeouts correctly, verify recovery, and handle the edge cases where automatic failover stalls.

## Normal Failover Sequence

Here is what happens when the Keeper leader fails:

```text
1. Leader goes down (crash, network, or shutdown)
2. Followers stop receiving heartbeats
3. After election_timeout (1-2s default), a follower becomes a candidate
4. Candidate requests votes from the other follower
5. Candidate becomes leader (wins a majority)
6. New leader starts sending heartbeats
7. ClickHouse clients reconnect to any Keeper node
8. The new leader begins forwarding accepted writes

Total time from failure to new leader: typically 2-5 seconds
```

## Simulating a Failover

Test your failover before a real outage:

```bash
# Step 1: Identify the current leader
for h in keeper1 keeper2 keeper3; do
    echo -n "${h}: "
    echo "stat" | nc ${h}.internal 2181 | grep Mode
done
# keeper1: Mode: leader
# keeper2: Mode: follower
# keeper3: Mode: follower

# Step 2: Stop the leader
ssh keeper1.internal "systemctl stop clickhouse-keeper"

# Step 3: Immediately start watching for a new leader
for h in keeper2 keeper3; do
    echo -n "${h}: "
    echo "stat" | nc ${h}.internal 2181 | grep Mode
done
# keeper2: Mode: leader     (elected within 1-2 seconds)
# keeper3: Mode: follower
```

## Verifying ClickHouse Replication Survives Failover

After the failover, verify ClickHouse is still replicating correctly:

```sql
-- Check that ClickHouse reconnected to Keeper
SELECT *
FROM system.zookeeper_connection;
-- Should show a valid connection to one of the Keeper nodes

-- Verify no replicas are in read-only mode
SELECT
    database,
    table,
    is_readonly,
    absolute_delay
FROM system.replicas
WHERE is_readonly = 1;
-- Should return 0 rows

-- Test an insert propagates
INSERT INTO events VALUES (today(), now(), 9001, 'failover_test', '{}');

-- Verify it replicated within a few seconds
SELECT count()
FROM clusterAllReplicas('production_cluster', currentDatabase(), events)
WHERE event_type = 'failover_test';
-- All replicas should return count > 0
```

## Configuring Failover Timeouts

The two key settings that control failover speed:

```xml
<coordination_settings>
    <!-- How often the leader sends heartbeats -->
    <heart_beat_interval_ms>500</heart_beat_interval_ms>

    <!-- A follower starts an election if it does not hear from the leader
         for a random duration in this range -->
    <election_timeout_lower_bound_ms>1000</election_timeout_lower_bound_ms>
    <election_timeout_upper_bound_ms>2000</election_timeout_upper_bound_ms>
</coordination_settings>
```

The randomized range prevents two followers from starting elections simultaneously (which would split votes and require additional election rounds). The range should be at least 2x the heartbeat interval.

For faster failover at the cost of false-positives (elections triggered by transient network hiccups):

```xml
<coordination_settings>
    <heart_beat_interval_ms>200</heart_beat_interval_ms>
    <election_timeout_lower_bound_ms>500</election_timeout_lower_bound_ms>
    <election_timeout_upper_bound_ms>1000</election_timeout_upper_bound_ms>
</coordination_settings>
```

For environments with unreliable network (virtualized infrastructure, cross-datacenter):

```xml
<coordination_settings>
    <heart_beat_interval_ms>1000</heart_beat_interval_ms>
    <election_timeout_lower_bound_ms>5000</election_timeout_lower_bound_ms>
    <election_timeout_upper_bound_ms>10000</election_timeout_upper_bound_ms>
</coordination_settings>
```

## ClickHouse Client Reconnection

ClickHouse automatically tries all Keeper nodes in the `<zookeeper>` config list when it loses a connection. Configure all Keeper nodes so that failover is transparent:

```xml
<!-- All three Keeper nodes in the config -->
<!-- ClickHouse tries them in order and moves to the next on failure -->
<zookeeper>
    <node>
        <host>keeper1.internal</host>
        <port>2181</port>
    </node>
    <node>
        <host>keeper2.internal</host>
        <port>2181</port>
    </node>
    <node>
        <host>keeper3.internal</host>
        <port>2181</port>
    </node>
    <!-- How long to wait before trying the next node -->
    <operation_timeout_ms>10000</operation_timeout_ms>
    <!-- Session timeout: replicas go read-only after this elapses without ZooKeeper -->
    <session_timeout_ms>30000</session_timeout_ms>
</zookeeper>
```

## When Automatic Failover Stalls

Failover can stall in these scenarios:

### Scenario 1: Only one node is alive

With 3 nodes and 2 down, the surviving node cannot form a quorum and will not accept writes. This is by design - Raft prevents data loss by refusing to elect a leader without a majority:

```bash
# The surviving node will show:
echo "stat" | nc keeper2.internal 2181 | grep Mode
# Mode: follower  (not leader, because no quorum)
```

There is no safe way to force a quorum with fewer than the required nodes without risking data loss. Bring back a second node.

### Scenario 2: Leader is alive but unreachable from ClickHouse

The cluster may have a leader but ClickHouse cannot connect to it. Check if the ClickHouse servers can reach all Keeper nodes:

```bash
# From a ClickHouse node
for h in keeper1 keeper2 keeper3; do
    echo -n "${h} port 2181: "
    nc -zv ${h}.internal 2181 && echo "OK" || echo "BLOCKED"
done
```

### Scenario 3: Election thrashing

If you see repeated elections in the log, the heartbeat timeout is too short for your network:

```bash
journalctl -u clickhouse-keeper -n 100 | grep -c "Become candidate"
# If this count is high over a short window, increase election timeouts
```

## Forcing a Leader Election

To manually trigger a failover (for example, to move the leader to a specific node for maintenance):

```bash
# Gracefully stop the current leader
# Keeper will elect a new leader from the remaining nodes
ssh keeper1.internal "systemctl stop clickhouse-keeper"

# Verify a new leader was elected
for h in keeper2 keeper3; do
    echo -n "${h}: "
    echo "stat" | nc ${h}.internal 2181 | grep Mode
done

# Restart the old leader as a follower
ssh keeper1.internal "systemctl start clickhouse-keeper"
```

## Keeper Recovery After Extended Downtime

If a Keeper node was offline while the cluster processed many operations, it will catch up by downloading a snapshot from the leader when it restarts:

```bash
# Restart the recovered node
systemctl start clickhouse-keeper

# Watch the log to see snapshot transfer
journalctl -u clickhouse-keeper -f | grep -i "snapshot\|sync\|follower"

# Expected output:
# [Raft] Downloading snapshot from leader
# [Raft] Snapshot downloaded, applying
# [Raft] Follower is now in sync with leader
```

After the snapshot transfer, the node rejoins the cluster as a follower and becomes available for client connections.
