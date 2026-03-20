# How to Deploy Redis Cluster via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Redis, Cluster, High Availability, Docker

Description: Learn how to deploy a Redis Cluster with multiple nodes via Portainer for high availability, horizontal scaling, and automatic failover.

## Redis Cluster Architecture

Redis Cluster distributes data across multiple nodes using hash slots. A minimum cluster requires 6 nodes: 3 primaries and 3 replicas.

```
Primary 1 (slots 0-5460)      ← Replica 1
Primary 2 (slots 5461-10922)  ← Replica 2
Primary 3 (slots 10923-16383) ← Replica 3
```

## Redis Cluster Stack via Portainer

**Stacks → Add Stack → redis-cluster**

```yaml
version: "3.8"

services:
  redis-1:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis1_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7001:7001"
      - "17001:17001"    # Cluster bus port (service port + 10000)

  redis-2:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis2_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7002:7001"
      - "17002:17001"

  redis-3:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis3_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7003:7001"
      - "17003:17001"

  redis-4:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis4_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7004:7001"
      - "17004:17001"

  redis-5:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis5_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7005:7001"
      - "17005:17001"

  redis-6:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - redis6_data:/data
      - ./redis-cluster.conf:/etc/redis/redis.conf:ro
    ports:
      - "7006:7001"
      - "17006:17001"

volumes:
  redis1_data:
  redis2_data:
  redis3_data:
  redis4_data:
  redis5_data:
  redis6_data:
```

## Redis Cluster Configuration File

```
# redis-cluster.conf
port 7001
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
requirepass your-cluster-password
masterauth your-cluster-password
bind 0.0.0.0
protected-mode no
```

## Initialize the Cluster

After deploying the stack, initialize the cluster from one of the nodes:

```bash
# Via Portainer exec on redis-1 container
redis-cli -a your-cluster-password \
  --cluster create \
  <HOST_IP>:7001 <HOST_IP>:7002 <HOST_IP>:7003 \
  <HOST_IP>:7004 <HOST_IP>:7005 <HOST_IP>:7006 \
  --cluster-replicas 1 \
  --cluster-yes
```

## Verify Cluster Status

```bash
redis-cli -a your-cluster-password \
  -h <HOST_IP> -p 7001 \
  cluster info

# Expected: cluster_state:ok, cluster_slots_assigned:16384
```

## Connecting Applications to Redis Cluster

```yaml
services:
  app:
    environment:
      # Most Redis clients support cluster mode URLs
      - REDIS_CLUSTER_NODES=host:7001,host:7002,host:7003
      - REDIS_PASSWORD=your-cluster-password
```

For Node.js (ioredis):

```javascript
const Redis = require('ioredis');
const cluster = new Redis.Cluster([
  { port: 7001, host: process.env.HOST_IP },
  { port: 7002, host: process.env.HOST_IP },
  { port: 7003, host: process.env.HOST_IP },
], {
  redisOptions: { password: process.env.REDIS_PASSWORD }
});
```

## Simpler Alternative: Redis Sentinel

For failover without data sharding, Redis Sentinel is simpler:

```yaml
services:
  redis-primary:
    image: redis:7.2-alpine

  redis-replica:
    image: redis:7.2-alpine
    command: redis-server --replicaof redis-primary 6379

  sentinel:
    image: redis:7.2-alpine
    command: redis-sentinel /etc/redis/sentinel.conf
```

## Conclusion

Redis Cluster via Portainer provides horizontal scaling and high availability for production cache workloads. For most small-to-medium applications, Redis Sentinel (primary + replica + sentinel) is simpler to configure and sufficient for failover needs. Use full cluster mode when you need data partitioning across multiple nodes for capacity or performance.
