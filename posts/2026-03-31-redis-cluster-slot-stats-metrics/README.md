# How to Use CLUSTER SLOT-STATS in Redis for Slot Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cluster, Monitoring

Description: Learn how to use CLUSTER SLOT-STATS in Redis to retrieve per-slot key count and memory usage metrics for load analysis and capacity planning.

---

`CLUSTER SLOT-STATS` is a Redis command introduced in Redis 7.4 that returns detailed statistics for each hash slot in a Redis Cluster. It provides key counts and memory usage data per slot, enabling fine-grained load analysis and capacity planning.

## Basic Syntax

```text
CLUSTER SLOT-STATS SLOTSRANGE start-slot end-slot
```

Or to filter by specific slots:

```text
CLUSTER SLOT-STATS SLOTS slot [slot ...]
```

## Checking Stats for a Range of Slots

```bash
# Get stats for slots 0 through 100
redis-cli -p 7001 CLUSTER SLOT-STATS SLOTSRANGE 0 100
```

Sample output:

```text
1) 1) (integer) 0
   2) 1) "key-count"
      2) (integer) 15
      3) "memory-usage-human"
      4) "1.20K"
2) 1) (integer) 1
   2) 1) "key-count"
      2) (integer) 3
      3) "memory-usage-human"
      4) "256B"
```

Each entry contains the slot number followed by its statistics map.

## Checking Stats for Specific Slots

```bash
redis-cli -p 7001 CLUSTER SLOT-STATS SLOTS 0 1000 8000 16383
```

This returns stats only for slots 0, 1000, 8000, and 16383.

## Finding Which Slot a Key Maps To

Before using SLOT-STATS, you may want to find which slot a specific key belongs to:

```bash
redis-cli CLUSTER KEYSLOT user:1001
# (integer) 4821
```

Then query stats for that slot:

```bash
redis-cli -p 7001 CLUSTER SLOT-STATS SLOTS 4821
```

## Using SLOT-STATS for Load Analysis

Uneven key distribution across slots can cause hot shards. Use SLOT-STATS to identify imbalances:

```bash
#!/bin/bash
# Find top 10 slots by key count across all slots
redis-cli -p 7001 CLUSTER SLOT-STATS SLOTSRANGE 0 16383 | \
  grep -A1 "key-count" | grep "integer" | sort -t: -k2 -rn | head -10
```

## Comparing Slot Distribution Across Nodes

In a multi-node cluster, run SLOT-STATS on each node to compare the distribution of keys assigned to that node's slots:

```bash
redis-cli -p 7001 CLUSTER SLOT-STATS SLOTSRANGE 0 5460
redis-cli -p 7002 CLUSTER SLOT-STATS SLOTSRANGE 5461 10922
redis-cli -p 7003 CLUSTER SLOT-STATS SLOTSRANGE 10923 16383
```

## Practical Capacity Planning

Use SLOT-STATS data to calculate average and peak memory per slot:

```python
import redis

client = redis.RedisCluster(host="127.0.0.1", port=7001)

# CLUSTER SLOT-STATS is a raw command - use execute_command
response = client.execute_command("CLUSTER SLOT-STATS SLOTSRANGE 0 16383")
key_counts = {}
for entry in response:
    slot = entry[0]
    stats = dict(zip(entry[1][::2], entry[1][1::2]))
    key_counts[slot] = int(stats.get(b"key-count", 0))

top_slots = sorted(key_counts.items(), key=lambda x: x[1], reverse=True)[:10]
for slot, count in top_slots:
    print(f"Slot {slot}: {count} keys")
```

## Differences from CLUSTER KEYSLOT and CLUSTER COUNTKEYSINSLOT

- `CLUSTER KEYSLOT key` - Returns the slot number for a single key.
- `CLUSTER COUNTKEYSINSLOT slot` - Returns the key count for one specific slot.
- `CLUSTER SLOT-STATS SLOTSRANGE start end` - Returns key count AND memory usage for a range of slots in one call, much more efficient for bulk analysis.

## Availability

`CLUSTER SLOT-STATS` requires Redis 7.4 or later. Check your version before using it:

```bash
redis-cli INFO server | grep redis_version
```

## Summary

`CLUSTER SLOT-STATS` provides per-slot key count and memory usage metrics for Redis Cluster nodes. It is the most efficient tool for identifying hot slots, auditing key distribution, and planning resharding operations. Use SLOTSRANGE for bulk queries and SLOTS for targeted slot inspection.
