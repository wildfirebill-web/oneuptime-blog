# How to Configure Redis with HAProxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, HAProxy, Load Balancer, High Availability, Proxy, Networking

Description: Set up HAProxy as a TCP proxy in front of Redis for connection multiplexing, health checking, and automatic failover between Redis primary and replica nodes.

---

## Why Use HAProxy with Redis

HAProxy provides:
- **TCP load balancing** - distribute Redis connections across multiple nodes
- **Health checks** - automatically remove unhealthy Redis nodes from the pool
- **Connection limiting** - protect Redis from connection overload
- **Stats endpoint** - monitor connection counts and backend health
- **Primary-only routing** - send writes only to the Redis primary

## Installing HAProxy

```bash
sudo apt update
sudo apt install -y haproxy
haproxy -v
# HAProxy version 2.8.x
```

## Basic Redis Proxy Configuration

Create or edit `/etc/haproxy/haproxy.cfg`:

```text
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 50000
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client  60s
    timeout server  60s

# HAProxy stats page
frontend stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:haproxy-stats-password

# Redis frontend
frontend redis_frontend
    bind *:6379
    mode tcp
    default_backend redis_backend

# Redis backend - primary only
backend redis_backend
    mode tcp
    balance leastconn
    option tcp-check
    tcp-check connect
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    server redis-primary 10.0.0.1:6379 check inter 3s
```

## Health Check That Verifies Primary Role

For Sentinel environments where the primary can change, add a health check that verifies the node is the primary before sending traffic:

```text
backend redis_backend
    mode tcp
    balance leastconn
    option tcp-check
    tcp-check connect
    tcp-check send AUTH\ yourpassword\r\n
    tcp-check expect string +OK
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK

    server redis-1 10.0.0.1:6379 check inter 3s fall 3 rise 2
    server redis-2 10.0.0.2:6379 check inter 3s fall 3 rise 2 backup
```

In this configuration:
- `redis-1` is the active server
- `redis-2` is the backup (used only if `redis-1` is marked down)
- HAProxy checks `role:master` in the INFO output to verify which is primary
- `fall 3` - mark down after 3 failed checks
- `rise 2` - mark up after 2 successful checks

## Separate Read and Write Ports

Expose Redis writes on port 6379 and reads on port 6380:

```text
# Write port - only routes to primary
frontend redis_write
    bind *:6379
    mode tcp
    default_backend redis_primary

backend redis_primary
    mode tcp
    option tcp-check
    tcp-check connect
    tcp-check send AUTH\ yourpassword\r\n
    tcp-check expect string +OK
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK

    server redis-1 10.0.0.1:6379 check inter 3s
    server redis-2 10.0.0.2:6379 check inter 3s backup

# Read port - distributes across replicas
frontend redis_read
    bind *:6380
    mode tcp
    default_backend redis_replicas

backend redis_replicas
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect
    tcp-check send PING\r\n
    tcp-check expect string +PONG

    server replica-1 10.0.0.3:6379 check inter 5s
    server replica-2 10.0.0.4:6379 check inter 5s
```

## Connection Limiting and Timeouts

Protect Redis from being overwhelmed:

```text
backend redis_backend
    mode tcp
    maxconn 5000          # Max connections to this backend
    timeout connect 1s    # Connection timeout to Redis
    timeout server 30s    # Idle server connection timeout

    server redis-1 10.0.0.1:6379 check inter 3s maxconn 1000
```

## Starting and Testing HAProxy

```bash
# Validate the config
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Start or restart HAProxy
sudo systemctl restart haproxy
sudo systemctl status haproxy

# Test Redis connectivity through HAProxy
redis-cli -h localhost -p 6379 -a yourpassword PING
# PONG

# Check HAProxy stats
curl -u admin:haproxy-stats-password http://localhost:8404/stats
```

## Monitoring HAProxy with the Stats Page

Access `http://haproxy-host:8404/stats` to view:
- Backend status (green = UP, red = DOWN)
- Session counts per server
- Connection rates
- Health check results

## HAProxy with Redis Sentinel

When using Redis Sentinel, configure HAProxy to re-check all potential primaries. After a Sentinel failover, the old backup becomes the new primary. HAProxy's health check with `role:master` verification will automatically route to the new primary:

```text
backend redis_sentinel_backend
    mode tcp
    balance first           # Always use the first UP server
    option tcp-check
    tcp-check connect
    tcp-check send AUTH\ yourpassword\r\n
    tcp-check expect string +OK
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK

    # All nodes are listed - only the master passes the health check
    server redis-1 10.0.0.1:6379 check inter 5s
    server redis-2 10.0.0.2:6379 check inter 5s
    server redis-3 10.0.0.3:6379 check inter 5s
```

With `balance first`, HAProxy always uses the first available server that passes the health check.

## HAProxy Logging

Configure HAProxy to log Redis connection events:

```text
global
    log /dev/log local0 info

defaults
    log global
    option tcplog
    log-format "%t %f %b/%s: %Tw/%Tc/%Tt %B %ts %ac/%fc/%bc/%sc/%rc %sq/%bq"
```

View logs:

```bash
sudo tail -f /var/log/syslog | grep haproxy
```

## Summary

HAProxy proxies Redis TCP connections, performs health checks that verify primary/replica role, and automatically routes traffic to the current primary during Sentinel failovers. Configure separate frontend/backend pairs for writes (port 6379) and reads (port 6380) to enable read scaling, use the `role:master` health check to ensure writes go only to the primary, and monitor backend health via the HAProxy stats page.
