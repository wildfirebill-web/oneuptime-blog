# How to Set Up Redis Cluster with Docker Compose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Docker Compose, High Availability, Sharding, DevOps

Description: Set up a Redis Cluster with 6 nodes using Docker Compose for local development and testing, with automatic slot assignment and cluster health verification.

---

## Redis Cluster Overview

Redis Cluster requires at least 3 primary nodes. A typical setup uses 6 nodes: 3 primaries and 3 replicas. Cluster distributes 16384 hash slots across the primaries.

## Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: "3.9"

x-redis-common: &redis-common
  image: redis:7-alpine
  restart: unless-stopped
  networks:
    - redis-cluster-net

services:
  redis-node-1:
    <<: *redis-common
    ports:
      - "7001:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  redis-node-2:
    <<: *redis-common
    ports:
      - "7002:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  redis-node-3:
    <<: *redis-common
    ports:
      - "7003:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  redis-node-4:
    <<: *redis-common
    ports:
      - "7004:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  redis-node-5:
    <<: *redis-common
    ports:
      - "7005:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  redis-node-6:
    <<: *redis-common
    ports:
      - "7006:6379"
    command: >
      redis-server
      --port 6379
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes

  cluster-init:
    image: redis:7-alpine
    networks:
      - redis-cluster-net
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
      - redis-node-4
      - redis-node-5
      - redis-node-6
    command: >
      sh -c 'sleep 5 && redis-cli --cluster create
      redis-node-1:6379 redis-node-2:6379 redis-node-3:6379
      redis-node-4:6379 redis-node-5:6379 redis-node-6:6379
      --cluster-replicas 1 --cluster-yes'
    restart: "no"

networks:
  redis-cluster-net:
    driver: bridge
```

## Start the Cluster

```bash
docker compose up -d
```

Wait for the `cluster-init` service to finish initializing the cluster:

```bash
docker compose logs cluster-init
```

You should see:
```text
[OK] All 16384 slots covered.
```

## Verify Cluster Status

```bash
# Check cluster health
redis-cli -p 7001 CLUSTER INFO

# View node list
redis-cli -p 7001 CLUSTER NODES
```

Expected output for `CLUSTER INFO`:

```text
cluster_enabled:1
cluster_state:ok
cluster_slots_assigned:16384
cluster_known_nodes:6
cluster_size:3
```

## Connect Using a Cluster Client

### Python

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host="127.0.0.1",
    port=7001,
    decode_responses=True
)

rc.set("hello", "world")
print(rc.get("hello"))

# Set multiple keys across different slots
for i in range(10):
    rc.set(f"key:{i}", f"value:{i}")

# Cluster MGET requires same-slot keys (use hash tags)
rc.set("{user}:name", "Alice")
rc.set("{user}:email", "alice@example.com")
print(rc.mget("{user}:name", "{user}:email"))
```

### Node.js

```javascript
const { Cluster } = require("ioredis");

const cluster = new Cluster([
  { host: "127.0.0.1", port: 7001 },
  { host: "127.0.0.1", port: 7002 },
  { host: "127.0.0.1", port: 7003 },
]);

async function main() {
  await cluster.set("foo", "bar");
  const val = await cluster.get("foo");
  console.log(val);
  await cluster.quit();
}

main();
```

## Checking Slot Distribution

```bash
redis-cli -p 7001 CLUSTER SHARDS
```

Or inspect per-node slot assignments:

```bash
redis-cli -p 7001 CLUSTER NODES | awk '{print $2, $9}'
```

## Teardown

```bash
docker compose down -v
```

## Summary

A Docker Compose Redis Cluster with 6 nodes (3 primaries, 3 replicas) is set up using a dedicated `cluster-init` service that calls `redis-cli --cluster create` after all nodes start. Cluster-aware clients like `redis-py` and `ioredis` handle slot routing and MOVED redirections automatically. Hash tags ensure related keys land on the same slot for multi-key operations.
