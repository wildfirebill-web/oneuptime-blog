# How to Use CLUSTER COUNTKEYSINSLOT in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cluster, CLUSTER COUNTKEYSINSLOT, Hash Slot, Key Count

Description: Learn how to use CLUSTER COUNTKEYSINSLOT to count the number of keys stored in a specific hash slot on the current Redis Cluster node.

---

`CLUSTER COUNTKEYSINSLOT` returns the number of keys in the local node's database that belong to a specified hash slot. This is useful for monitoring slot utilization, verifying data migration, and understanding key distribution across your cluster.

## Syntax

```text
CLUSTER COUNTKEYSINSLOT slot
```

Returns an integer - the count of keys in that slot on the current node.

## Basic Usage

```bash
# Count keys in slot 0
redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT 0

# Count keys in slot 500
redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT 500
# (integer) 42
```

## Checking Slot Utilization Across the Cluster

To understand data distribution, check key counts across multiple slots:

```bash
for slot in $(seq 0 100 1000); do
  count=$(redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT $slot)
  echo "Slot $slot: $count keys"
done
```

## Verifying Data Migration

During a slot migration from node A to node B, use `CLUSTER COUNTKEYSINSLOT` on both nodes to track progress:

```bash
# Check source node (should decrease)
redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT 500

# Check destination node (should increase)
redis-cli -p 7002 CLUSTER COUNTKEYSINSLOT 500
```

Migration is complete when the source shows 0 and all keys appear on the destination.

## Locating Empty vs Populated Slots

Identify which slots have data and which are empty:

```bash
for slot in $(seq 0 16383); do
  count=$(redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT $slot)
  if [ "$count" -gt 0 ]; then
    echo "Slot $slot: $count keys"
  fi
done
```

This is slow for a full scan - use it on a subset of slots or only during maintenance.

## Pairing with CLUSTER GETKEYSINSLOT

`CLUSTER COUNTKEYSINSLOT` tells you how many keys exist; `CLUSTER GETKEYSINSLOT` lets you retrieve them:

```bash
# How many keys are in slot 500?
count=$(redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT 500)
echo "Keys in slot 500: $count"

# Retrieve up to 10 key names
redis-cli -p 7001 CLUSTER GETKEYSINSLOT 500 10
```

## Monitoring Slot Balance

An uneven key distribution can cause hot spots. Use this check to find overloaded slots:

```bash
#!/bin/bash
max=0
max_slot=0
for slot in $(seq 0 16383); do
  count=$(redis-cli -p 7001 CLUSTER COUNTKEYSINSLOT $slot)
  if [ "$count" -gt "$max" ]; then
    max=$count
    max_slot=$slot
  fi
done
echo "Most loaded slot: $max_slot with $max keys"
```

## Summary

`CLUSTER COUNTKEYSINSLOT` provides a quick count of keys in any hash slot on the current node. It is invaluable during migration verification, distribution audits, and slot balance analysis. Combine it with `CLUSTER GETKEYSINSLOT` and `CLUSTER KEYSLOT` for a complete picture of your cluster's key distribution.
