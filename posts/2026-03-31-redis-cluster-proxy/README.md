# How to Scale Redis with Redis Cluster Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster Proxy, Redis Cluster, Scalability, Proxy

Description: Use Redis Cluster Proxy to expose a Redis Cluster as a single endpoint, allowing non-cluster-aware clients to transparently interact with a sharded cluster.

---

Redis Cluster Proxy (`redis-cluster-proxy`) bridges the gap between non-cluster-aware Redis clients and a Redis Cluster. Instead of requiring your application to handle MOVED/ASK redirects or use cluster-aware client libraries, the proxy presents a single Redis endpoint that internally routes commands to the correct cluster node.

## Why Use Redis Cluster Proxy

Many legacy applications use basic Redis clients that do not understand the Redis Cluster protocol. Redis Cluster Proxy lets you migrate these applications to a clustered setup without rewriting your client code.

## Building and Installing

```bash
git clone https://github.com/RedisLabs/redis-cluster-proxy.git
cd redis-cluster-proxy
make
sudo cp src/redis-cluster-proxy /usr/local/bin/
```

## Starting the Proxy

Point the proxy at any node in your cluster:

```bash
redis-cluster-proxy \
  --port 7777 \
  --cluster 127.0.0.1:7001 \
  --threads 8 \
  --max-clients 10000 \
  --log-level verbose
```

Key flags:
- `--cluster` - entry point to the Redis Cluster
- `--threads` - number of IO threads (set to CPU count for best performance)
- `--max-clients` - maximum simultaneous client connections

## Connecting a Non-Cluster-Aware Client

```python
import redis

# Connect to the proxy as if it were a standalone Redis
client = redis.Redis(host="localhost", port=7777, decode_responses=True)

# Commands are transparently routed to the correct cluster node
client.set("user:100:name", "Bob")
value = client.get("user:100:name")
print(value)  # Bob

# Pipeline works too
pipe = client.pipeline()
pipe.set("key1", "val1")
pipe.set("key2", "val2")
pipe.execute()
```

## Multi-Key Command Handling

Redis Cluster Proxy handles cross-slot multi-key commands by splitting them internally:

```bash
# This works through the proxy even if keys span multiple slots
redis-cli -p 7777 MGET user:1:name user:2:name user:3:name
```

Note: some cross-slot commands like MSET and transactions (MULTI/EXEC) require all keys to map to the same slot for atomic execution. Use hash tags to enforce colocation.

## Monitoring the Proxy

Redis Cluster Proxy exposes proxy-specific info:

```bash
redis-cli -p 7777 PROXY INFO
redis-cli -p 7777 PROXY CLUSTERS
redis-cli -p 7777 PROXY STATS
```

Sample output:

```text
cluster_ok:1
connected_clients:142
total_commands_processed:4817293
proxy_threads:8
```

## Running as a Systemd Service

```text
[Unit]
Description=Redis Cluster Proxy
After=network.target

[Service]
ExecStart=/usr/local/bin/redis-cluster-proxy --port 7777 --cluster 127.0.0.1:7001 --threads 8
Restart=always
User=redis

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable redis-cluster-proxy
sudo systemctl start redis-cluster-proxy
```

## Summary

Redis Cluster Proxy enables non-cluster-aware clients to work with a Redis Cluster by handling all routing, MOVED/ASK redirects, and cross-slot command splitting internally. It is ideal for migrating existing applications to Redis Cluster without a client library upgrade. Monitor proxy stats and tune thread count for your workload to maximize throughput.
