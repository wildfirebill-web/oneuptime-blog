# How to Test ClickHouse Failover Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Failover, Testing, High Availability, Operation

Description: Learn how to test ClickHouse failover procedures in a controlled way to validate your cluster's resilience before a real outage occurs.

---

## Why Test Failover?

Untested failover procedures are unreliable. Configurations can drift, replicas can fall behind, and runbooks can become outdated. Regular failover tests ensure your cluster can actually recover and that your team knows exactly what to do.

## Pre-Test Checklist

Before running any failover test, confirm:

```sql
-- Check replicas are in sync
SELECT replica_name, is_readonly, queue_size,
       log_max_index - log_pointer AS lag
FROM system.replicas;

-- Verify all shards are healthy
SELECT * FROM system.clusters WHERE cluster = 'prod_cluster';

-- Check ZooKeeper/Keeper connectivity
SELECT * FROM system.zookeeper WHERE path = '/';
```

## Test 1: Single Replica Failure

Simulate a single node going down by stopping the ClickHouse service:

```bash
# On the node you want to test
sudo systemctl stop clickhouse-server

# From another node, verify the cluster still serves queries
clickhouse-client --host ch-node-2 \
  --query "SELECT count() FROM events_distributed"
```

Expected: queries succeed; the Distributed table skips the failed replica.

## Test 2: Primary Node Failure with HAProxy

```bash
# Stop the primary
sudo systemctl stop clickhouse-server  # on ch-primary

# Verify HAProxy detects the failure (check logs)
tail -f /var/log/haproxy.log | grep "ch-primary"

# Run queries through the load balancer
for i in {1..5}; do
  curl -s "http://haproxy:8123/?query=SELECT+hostName()"
done
```

Expected: all requests return the standby node's hostname.

## Test 3: ZooKeeper Leader Failure

```bash
# Find the ZooKeeper leader
echo "stat" | nc zk-node-1 2181 | grep Mode

# Stop the leader
sudo systemctl stop zookeeper  # on the leader node

# Verify ClickHouse still serves queries (reads should work; writes may pause briefly)
clickhouse-client --query "SELECT count() FROM events_local"
```

Expected: ClickHouse continues serving reads. Writes may pause for a few seconds while ZooKeeper elects a new leader.

## Test 4: Network Partition Simulation

```bash
# Block traffic from ch-node-1 to ch-node-2
sudo iptables -A INPUT -s ch-node-2-ip -j DROP
sudo iptables -A OUTPUT -d ch-node-2-ip -j DROP

# Check system.replicas on ch-node-1
clickhouse-client --host ch-node-1 \
  --query "SELECT * FROM system.replicas WHERE table = 'events_local'"

# Restore connectivity
sudo iptables -D INPUT -s ch-node-2-ip -j DROP
sudo iptables -D OUTPUT -d ch-node-2-ip -j DROP
```

## Post-Recovery Verification

After restoring a failed node, verify data consistency:

```sql
-- Check replica caught up
SYSTEM SYNC REPLICA events_local;

-- Compare row counts
SELECT count() FROM events_local;  -- on both nodes

-- Check for diverged parts
SELECT * FROM system.replicas WHERE parts_to_check > 0;
```

## Documenting Results

For each test, record:

```text
Test:          Single replica failure
Date:          2026-03-31
Duration:      15 minutes
Impact:        0 query errors (Distributed table failed over automatically)
Recovery time: 8 seconds after node restart
Action items:  None - within SLA
```

## Summary

Regular failover testing is essential for maintaining confidence in your ClickHouse cluster's resilience. Test single replica failures, primary node failures through your load balancer, and ZooKeeper leader elections. Document results and track recovery times over time to detect configuration drift before it becomes a real outage.
