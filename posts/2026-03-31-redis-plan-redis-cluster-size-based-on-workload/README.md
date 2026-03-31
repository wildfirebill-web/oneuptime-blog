# How to Plan Redis Cluster Size Based on Workload

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Capacity Planning, Scalability, DevOps

Description: Learn how to size a Redis Cluster by estimating memory, throughput, and shard count requirements based on your actual workload data and growth projections.

---

Redis Cluster distributes data across multiple shards (primary nodes), each responsible for a subset of 16,384 hash slots. Sizing a cluster correctly from the start avoids painful resharding during high traffic periods.

## Step 1: Measure Current Workload

Before sizing, gather metrics from your existing Redis or staging environment:

```bash
# Key metrics to collect
redis-cli INFO memory | grep used_memory_human
redis-cli INFO stats | grep instantaneous_ops_per_sec
redis-cli INFO clients | grep connected_clients
redis-cli INFO replication | grep role
redis-cli INFO keyspace
```

Capture these values during peak traffic. You need:

- **Peak used memory** (GB)
- **Peak ops per second** (commands/sec)
- **Peak connected clients**
- **Read vs write ratio** (estimate from your application)

## Step 2: Calculate Required Shard Count for Memory

Each shard should use no more than 70-75% of its allocated memory, leaving headroom for replication buffers, temporary key growth, and fragmentation:

```python
def required_shards_for_memory(
    total_data_gb: float,
    memory_per_node_gb: float,
    headroom_factor: float = 1.35  # 35% headroom
) -> dict:
    effective_memory = memory_per_node_gb / headroom_factor
    num_shards = max(3, int(total_data_gb / effective_memory) + 1)
    return {
        "total_data_gb": total_data_gb,
        "memory_per_node_gb": memory_per_node_gb,
        "effective_memory_per_shard_gb": round(effective_memory, 2),
        "min_shards": num_shards,
    }

# 50 GB of data, 16 GB nodes
print(required_shards_for_memory(50, 16))
# {'total_data_gb': 50, 'memory_per_node_gb': 16,
#  'effective_memory_per_shard_gb': 11.85, 'min_shards': 5}
```

Minimum shard count is 3 (required by Redis Cluster for quorum).

## Step 3: Calculate Required Shard Count for Throughput

A single Redis shard typically handles 100,000-200,000 operations per second before CPU becomes the bottleneck:

```python
def required_shards_for_throughput(
    peak_ops_per_sec: int,
    max_ops_per_shard: int = 100_000,
    safety_factor: float = 0.7  # use 70% of max capacity
) -> dict:
    effective_capacity = max_ops_per_shard * safety_factor
    num_shards = max(3, int(peak_ops_per_sec / effective_capacity) + 1)
    return {
        "peak_ops_per_sec": peak_ops_per_sec,
        "effective_capacity_per_shard": int(effective_capacity),
        "min_shards": num_shards,
    }

# 500,000 ops/sec peak
print(required_shards_for_throughput(500_000))
# {'peak_ops_per_sec': 500000, 'effective_capacity_per_shard': 70000, 'min_shards': 8}
```

## Step 4: Choose the Final Shard Count

Take the maximum of memory and throughput requirements, then plan for replicas:

```bash
# Final sizing decision
MEMORY_SHARDS=5   # from memory calculation
THROUGHPUT_SHARDS=8   # from throughput calculation
SHARDS=$(( THROUGHPUT_SHARDS > MEMORY_SHARDS ? THROUGHPUT_SHARDS : MEMORY_SHARDS ))
REPLICAS=1  # standard: 1 replica per primary

echo "Primary shards: $SHARDS"
echo "Total nodes:    $((SHARDS * (1 + REPLICAS)))"
```

For a cluster with 8 shards and 1 replica per shard: **16 total nodes**.

## Step 5: Provision the Cluster

```bash
# Create a 3-shard cluster (minimum viable) for development
redis-cli --cluster create \
  10.0.0.1:6379 10.0.0.2:6379 10.0.0.3:6379 \
  10.0.0.4:6379 10.0.0.5:6379 10.0.0.6:6379 \
  --cluster-replicas 1

# Verify cluster state
redis-cli --cluster check 10.0.0.1:6379
```

## Step 6: Monitor Cluster Balance

After provisioning, verify hash slots are distributed evenly:

```bash
redis-cli -c -h 10.0.0.1 CLUSTER INFO | grep cluster_size
redis-cli -c -h 10.0.0.1 CLUSTER NODES | awk '{print $3, $9}' | grep master
```

Each shard should own approximately `16384 / num_shards` slots.

## Adding Shards When You Grow

```bash
# Add a new shard node pair and rebalance
redis-cli --cluster add-node 10.0.0.7:6379 10.0.0.1:6379
redis-cli --cluster add-node 10.0.0.8:6379 10.0.0.1:6379 --cluster-slave --cluster-master-id <master-id>
redis-cli --cluster rebalance 10.0.0.1:6379
```

## Summary

Redis Cluster sizing requires calculating shard count from both memory requirements (data / (node_memory * 0.7)) and throughput requirements (peak_ops / 70,000). Take the higher of the two, maintain at least 3 shards for cluster consensus, and add 1 replica per shard for HA. Monitor slot balance and per-shard memory in OneUptime to catch imbalances before they affect performance.
