# How to Configure Redis TCP Backlog and Connection Limits

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Configuration, TCP, Connection Limits, Performance

Description: Learn how to configure Redis TCP backlog and connection limits to handle high-concurrency workloads and prevent connection queue overflow.

---

## What Is the TCP Backlog in Redis?

The TCP backlog is the queue of pending connections waiting to be accepted by the Redis server. When many clients connect simultaneously, connections that cannot be immediately accepted are held in this queue. If the queue fills up, new connection attempts are dropped.

The Redis `tcp-backlog` directive sets the size of this queue.

## Default Configuration

```text
# redis.conf default
tcp-backlog 511
```

Redis requests a backlog of 511 connections from the OS. The actual value may be lower if the OS kernel limit (`/proc/sys/net/core/somaxconn`) is smaller.

## Setting TCP Backlog

```text
# redis.conf

# High-traffic production setup
tcp-backlog 4096

# Default for moderate traffic
tcp-backlog 511

# Minimal (not recommended for production)
tcp-backlog 128
```

## System Kernel Requirement

The OS must also support the larger backlog size. Check and increase it if needed:

```bash
# Check current kernel limit
cat /proc/sys/net/core/somaxconn
# 128 (common default - may limit Redis backlog)

# Increase for the current session
sudo sysctl -w net.core.somaxconn=4096

# Make permanent
echo "net.core.somaxconn=4096" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Also check the TCP SYN backlog:

```bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
# 256 (increase for high connection rates)

sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096
```

## Maximum Client Connections

Use `maxclients` to limit the total number of simultaneous connected clients:

```text
# redis.conf

# Allow up to 10000 simultaneous clients
maxclients 10000

# Default is 10000 (adjustable based on ulimit)
```

Redis file descriptor limit must be high enough:

```bash
# Check current file descriptor limit
ulimit -n
# 1024 (too low for high-connection Redis)

# Increase for current session
ulimit -n 65535

# Persistent change in /etc/security/limits.conf
redis soft nofile 65535
redis hard nofile 65535
```

## Redis systemd Service File

For Redis running under systemd, set file descriptor limits:

```text
# /etc/systemd/system/redis.service
[Service]
LimitNOFILE=65535
```

```bash
# Reload systemd and restart Redis
sudo systemctl daemon-reload
sudo systemctl restart redis
```

## Monitoring Connection Stats

```bash
# Check current connected clients
redis-cli INFO clients

# Key fields:
# connected_clients: currently connected clients
# blocked_clients: clients blocked on BLPOP/BRPOP
# tracking_clients: clients using CLIENT TRACKING
# clients_in_timeout_table: clients pending timeout
```

```bash
# Check if connections are being dropped
redis-cli INFO stats | grep rejected
# rejected_connections: 42  (non-zero = clients being turned away)
```

## Python Example: Connection Pool Configuration

For applications, use a connection pool sized appropriately:

```python
import redis

# Configure connection pool for high-concurrency app
pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    max_connections=100,      # Max pool size
    socket_connect_timeout=5, # Connection timeout in seconds
    socket_timeout=10,        # Command timeout in seconds
    retry_on_timeout=True
)

r = redis.Redis(connection_pool=pool)

def check_pool_stats():
    """Check connection pool health."""
    in_use = len(pool._in_use_connections)
    available = pool.max_connections - in_use
    print(f"Pool: {in_use} in use, {available} available of {pool.max_connections}")

check_pool_stats()
```

## Full Production Configuration Example

```text
# redis.conf - high-traffic production

# Network
bind 127.0.0.1 10.0.1.5
port 6379
tcp-backlog 4096
tcp-keepalive 300

# Connection limits
maxclients 10000
timeout 300

# Memory
maxmemory 4gb
maxmemory-policy allkeys-lru

# Logging
loglevel notice
logfile /var/log/redis/redis.log
```

## Diagnosing Connection Drops

If clients see `Connection refused` or `ECONNRESET`:

```bash
# Check if backlog is filling
ss -lnt | grep 6379
# Recv-Q column shows pending connections

# Check rejected_connections in stats
watch -n 1 "redis-cli INFO stats | grep rejected_connections"

# Check OS kernel backlog limit
cat /proc/sys/net/core/somaxconn
```

## Summary

The `tcp-backlog` directive controls how many pending connection attempts Redis queues. In high-traffic environments, increase it to 4096 and ensure the kernel's `net.core.somaxconn` is set equally high. Pair with `maxclients` to set a hard cap on simultaneous connections and set the OS file descriptor limit (`nofile`) to support the desired connection count. Monitor `rejected_connections` in Redis stats to detect when clients are being turned away.
