# How to Handle Split-Brain Scenarios in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Split-Brain, High Availability, Sentinel, Cluster

Description: Understand Redis split-brain scenarios, when they occur in Sentinel and Cluster setups, and how to configure safeguards to prevent data inconsistency.

---

A split-brain occurs when two nodes in a distributed Redis setup believe they are both the primary, leading to conflicting writes and data divergence. Understanding when this happens and how Redis mitigates it is essential for production reliability.

## When Split-Brain Occurs

Split-brain typically happens during network partitions. If a primary becomes unreachable to Sentinels but can still accept client writes, a new primary may be elected while the old one continues serving writes.

```text
Before partition:
  Clients -> Primary -> Replica-1, Replica-2
  Sentinels monitor all nodes

During partition:
  Partition A: Clients + Old Primary (still accepting writes)
  Partition B: Sentinels + Replica-1 -> elected as New Primary
```

After the partition heals, the old primary reconnects as a replica and discards its writes.

## Redis Sentinel's Built-In Protection

Sentinel's `min-replicas-to-write` and `min-replicas-max-lag` settings limit writes on an isolated primary:

```bash
# In redis.conf on the primary
min-replicas-to-write 1
min-replicas-max-lag 10
```

With this config, the primary stops accepting writes if fewer than 1 replica has acknowledged within 10 seconds. This limits data loss during a split-brain but does not eliminate it entirely.

## Configuring Sentinel Quorum Carefully

Ensure your quorum is set to a majority of Sentinels to avoid false failovers:

```bash
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
# With 3 Sentinels, quorum=2 requires majority agreement
```

Avoid quorum=1 with 3 Sentinels - it makes false failovers too easy.

## Detecting Split-Brain After the Fact

When the old primary rejoins as a replica, Redis logs the demotion:

```text
REPLICAOF 192.168.1.20 6379 called from Sentinel, discarding replication buffer: ...
```

Check replication state immediately after network recovery:

```bash
redis-cli -h old-primary INFO replication | grep role
# role:slave confirms demotion
```

## Data Recovery After Split-Brain

Writes made to the isolated old primary are lost when it rejoins. To minimize this:

```python
# Use idempotent operations where possible
# Instead of INCR (which may be applied twice):
redis.incr('counter')

# Prefer SET with a version check via Lua
lua_script = """
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
  return redis.call('SET', KEYS[1], ARGV[2])
end
return 0
"""
```

## Redis Cluster Split-Brain Handling

In Redis Cluster, a shard becomes unavailable if it loses its primary and cannot elect a new one (no replica or no cluster quorum). The cluster prevents writes to affected slots:

```bash
redis-cli CLUSTER INFO | grep cluster_state
# cluster_state:ok   (normal)
# cluster_state:fail (split-brain or node loss)
```

Configure `cluster-require-full-coverage no` to allow the cluster to serve available slots even if some are down:

```bash
# redis.conf
cluster-require-full-coverage no
```

## Monitoring for Split-Brain

Alert when you see multiple primaries for the same shard or when `master_last_io_seconds_ago` jumps suddenly:

```bash
redis-cli INFO replication | grep master_last_io_seconds_ago
```

Use OneUptime to monitor Redis health endpoints and alert on abnormal replication state changes.

## Summary

Redis split-brain is managed through `min-replicas-to-write` settings on primaries, careful Sentinel quorum configuration, and Cluster state monitoring. While Redis cannot fully prevent split-brain during partitions, these controls minimize data loss and ensure fast recovery when connectivity is restored.
