# How to Configure Redis Cluster with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis Cluster, IPv6, Sharding, High Availability, Distributed Cache, Redis

Description: Configure a Redis Cluster with IPv6 nodes for horizontal scaling and high availability, covering node configuration, cluster creation, and client connectivity over IPv6.

---

Redis Cluster provides automatic data sharding across multiple Redis nodes. Configuring it with IPv6 requires setting IPv6 bind addresses and cluster announce addresses in each node's configuration.

## Redis Cluster Node Configuration

Each Redis cluster node needs its own configuration file:

```bash
# /etc/redis/redis-cluster-7001.conf (Node 1)

port 7001
cluster-enabled yes
cluster-config-file nodes-7001.conf
cluster-node-timeout 5000

# Bind to IPv6

bind 2001:db8::1 ::1

# Announce this node's IPv6 address to other cluster members
cluster-announce-ip 2001:db8::1
cluster-announce-port 7001
cluster-announce-bus-port 17001

# Data persistence
appendonly yes
appendfilename "appendonly-7001.aof"
dir /var/lib/redis/cluster/7001/

# Log file
logfile "/var/log/redis/cluster-7001.log"
```

Create similar configs for all 6 nodes (3 primaries + 3 replicas):

```bash
# Create configs for all 6 nodes
for port in 7001 7002 7003 7004 7005 7006; do
  mkdir -p /var/lib/redis/cluster/$port
  cat > /etc/redis/redis-cluster-$port.conf << EOF
port $port
cluster-enabled yes
cluster-config-file nodes-$port.conf
cluster-node-timeout 5000
bind :: ::1
cluster-announce-ip 2001:db8::$(( (port - 7000) ))
cluster-announce-port $port
cluster-announce-bus-port 1$port
appendonly yes
dir /var/lib/redis/cluster/$port/
logfile /var/log/redis/cluster-$port.log
EOF
done
```

## Starting Redis Cluster Nodes

```bash
# Start all 6 Redis instances
for port in 7001 7002 7003 7004 7005 7006; do
  redis-server /etc/redis/redis-cluster-$port.conf --daemonize yes
done

# Verify all are listening on IPv6
ss -tlnp | grep "70[0-9][0-9]"
```

## Creating the Redis Cluster with IPv6

```bash
# Create cluster - assign primaries and replicas
# --cluster-replicas 1 = 1 replica per primary
redis-cli --cluster create \
  [2001:db8::1]:7001 \
  [2001:db8::2]:7002 \
  [2001:db8::3]:7003 \
  [2001:db8::4]:7004 \
  [2001:db8::5]:7005 \
  [2001:db8::6]:7006 \
  --cluster-replicas 1 \
  --cluster-yes

# Verify cluster was created
redis-cli -h 2001:db8::1 -p 7001 CLUSTER INFO
redis-cli -h 2001:db8::1 -p 7001 CLUSTER NODES
```

## Testing Redis Cluster over IPv6

```bash
# Connect in cluster mode
redis-cli -h 2001:db8::1 -p 7001 -c

# Set key (will redirect to correct shard if needed)
redis-cli -c -h 2001:db8::1 -p 7001 SET testkey "ipv6cluster"
redis-cli -c -h 2001:db8::1 -p 7001 GET testkey

# Test with multiple keys
redis-cli -c -h 2001:db8::1 -p 7001 SET key1 val1
redis-cli -c -h 2001:db8::1 -p 7001 SET key2 val2
redis-cli -c -h 2001:db8::1 -p 7001 SET key3 val3
```

## Python Redis Cluster Client over IPv6

```python
from redis.cluster import RedisCluster
from redis.cluster import ClusterNode

# Define startup nodes with IPv6 addresses
startup_nodes = [
    ClusterNode('2001:db8::1', 7001),
    ClusterNode('2001:db8::2', 7002),
    ClusterNode('2001:db8::3', 7003),
]

# Create cluster client
rc = RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    skip_full_coverage_check=True
)

# Use cluster
rc.set('ipv6_key', 'ipv6_value')
value = rc.get('ipv6_key')
print(f"Value: {value}")

# Close connection
rc.close()
```

## Adding Nodes to the Cluster

```bash
# Add a new primary node
redis-cli --cluster add-node \
  [2001:db8::7]:7007 \
  [2001:db8::1]:7001

# Add a replica node
redis-cli --cluster add-node \
  [2001:db8::8]:7008 \
  [2001:db8::1]:7001 \
  --cluster-slave \
  --cluster-master-id <primary-node-id>

# Rebalance slots
redis-cli --cluster rebalance [2001:db8::1]:7001
```

## Cluster Health Checks

```bash
# Check cluster health
redis-cli --cluster check [2001:db8::1]:7001

# Get cluster info
redis-cli -h 2001:db8::1 -p 7001 CLUSTER INFO | grep -E "cluster_state|slots"

# View all nodes
redis-cli -h 2001:db8::1 -p 7001 CLUSTER NODES
```

Redis Cluster with IPv6 announce addresses enables building horizontally scalable distributed cache infrastructure on IPv6 networks, maintaining the same data sharding and replication guarantees as IPv4 clusters.
