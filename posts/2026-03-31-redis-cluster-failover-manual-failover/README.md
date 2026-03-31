# How to Use CLUSTER FAILOVER in Redis for Manual Failover

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cluster, CLUSTER FAILOVER, High Availability, Replication

Description: Learn how to use CLUSTER FAILOVER to promote a replica to primary in Redis Cluster for planned maintenance and controlled failovers.

---

`CLUSTER FAILOVER` is issued on a replica node to initiate a manual failover, promoting it to primary. Unlike an automatic failover triggered by node failure, this command gives you a controlled, coordinated way to switch leadership - useful for planned maintenance, rolling upgrades, or load rebalancing.

## Syntax

```text
CLUSTER FAILOVER [FORCE | TAKEOVER]
```

Run this command on the **replica** you want to promote.

## Failover Modes

| Mode | Description |
|------|-------------|
| (default) | Negotiated - replica requests the primary stop accepting writes, completes replication sync, then takes over |
| FORCE | Skips negotiation - promotes even if primary is unreachable |
| TAKEOVER | Overrides cluster consensus entirely - use only in emergencies |

## Default Failover - Planned Maintenance

The safest option. The replica coordinates with the primary to ensure zero data loss:

```bash
# Connect to the replica (port 7004) and run failover
redis-cli -p 7004 CLUSTER FAILOVER
```

Redis will:
1. Tell the primary to stop accepting writes
2. Wait for the replica to fully catch up
3. Promote the replica to primary
4. Update cluster topology

## Monitoring Failover Progress

```bash
# Watch the role change
redis-cli -p 7004 CLUSTER INFO
# cluster_state: ok
# Look for the node transitioning from slave to master

redis-cli -p 7004 ROLE
# ["master", 0, [...]]
```

## FORCE Mode - Primary Unreachable

If the primary is down and automatic failover has not occurred, use FORCE:

```bash
redis-cli -p 7004 CLUSTER FAILOVER FORCE
```

This may result in a brief period of data loss if the replica was not fully synced.

## TAKEOVER Mode - Emergency Only

TAKEOVER immediately promotes the replica without any coordination:

```bash
redis-cli -p 7004 CLUSTER FAILOVER TAKEOVER
```

Use this only when the cluster is severely degraded and other failover methods fail. It can cause split-brain if the original primary recovers.

## Practical Use Case - Rolling Upgrade

When upgrading primary nodes without downtime:

```bash
# Step 1 - Promote replica to become new primary
redis-cli -p 7004 CLUSTER FAILOVER

# Step 2 - Wait for failover to complete
redis-cli -p 7004 ROLE

# Step 3 - Upgrade the old primary (now a replica)
# Perform upgrade on port 7001...

# Step 4 - Optionally fail back
redis-cli -p 7001 CLUSTER FAILOVER
```

## Summary

`CLUSTER FAILOVER` enables planned, controlled promotion of a Redis Cluster replica to primary. The default mode is safest and ensures no data loss by coordinating with the primary. `FORCE` and `TAKEOVER` modes are available for scenarios where the primary is unavailable, with increasing levels of risk. Always prefer the default mode for routine maintenance.
