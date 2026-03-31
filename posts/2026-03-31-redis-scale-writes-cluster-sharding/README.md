# How to Scale Redis Writes with Cluster Sharding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Sharding, Scalability, Performance

Description: Scale Redis write throughput beyond a single node using Redis Cluster, which automatically shards data across multiple primary nodes using hash slots.

---

A single Redis primary can saturate under heavy write loads - typically around 100,000 to 200,000 writes per second. Redis Cluster distributes data across multiple primary nodes using hash slots, letting you scale writes horizontally.

## How Redis Cluster Sharding Works

Redis Cluster divides the key space into 16,384 hash slots. Each primary node owns a range of slots. When a client writes a key, Redis computes `CRC16(key) % 16384` to determine which slot (and therefore which node) handles that key.

## Setting Up a 3-Node Cluster

Create six `redis.conf` files (3 primaries + 3 replicas):

```text
port 7001
cluster-enabled yes
cluster-config-file nodes-7001.conf
cluster-node-timeout 5000
appendonly yes
```

Start the nodes:

```bash
for port in 7001 7002 7003 7004 7005 7006; do
  redis-server /etc/redis/redis-$port.conf --daemonize yes
done
```

Create the cluster with replication:

```bash
redis-cli --cluster create \
  127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 \
  127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 \
  --cluster-replicas 1
```

## Verifying Cluster Health

```bash
redis-cli -p 7001 CLUSTER INFO
redis-cli -p 7001 CLUSTER NODES
```

Check `cluster_state:ok` and that all 16384 slots are covered.

## Writing to the Cluster in Python

```python
from redis.cluster import RedisCluster, ClusterNode

startup_nodes = [
    ClusterNode("127.0.0.1", 7001),
    ClusterNode("127.0.0.1", 7002),
    ClusterNode("127.0.0.1", 7003),
]

client = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)

# Writes are automatically routed to the correct shard
for i in range(10000):
    client.set(f"user:{i}:name", f"User {i}", ex=3600)

print("Written 10,000 keys across cluster")
```

## Hash Tags for Multi-Key Operations

Keys with hash tags are colocated on the same slot, enabling multi-key commands:

```python
# {user:42} forces both keys to the same slot
client.set("{user:42}:name", "Alice")
client.set("{user:42}:email", "alice@example.com")

# MGET works because both keys share the same slot
values = client.mget("{user:42}:name", "{user:42}:email")
```

## Rebalancing Slots After Adding a Node

```bash
# Add a new node
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7001

# Rebalance slots evenly
redis-cli --cluster rebalance 127.0.0.1:7001
```

## Monitoring Write Distribution

Check how writes are distributed across shards:

```bash
for port in 7001 7002 7003; do
  echo "Node :$port"
  redis-cli -p $port INFO stats | grep total_commands_processed
done
```

## Summary

Redis Cluster sharding scales write throughput by distributing keys across multiple primary nodes using hash slots. Setting up a cluster requires at least three primaries, and using hash tags ensures colocated keys for multi-key operations. As write load grows, adding nodes and rebalancing slots keeps performance consistent without downtime.
