# How to Configure D3N Redis Backend for RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, D3N, Redis, RGW, Cache, Object Storage

Description: Configure Redis as the distributed cache backend for D3N in Ceph RGW to coordinate cache state across multiple RGW instances in a zone.

---

## Overview

When running multiple RGW instances in a single zone, D3N needs a shared cache index so all instances know which objects are cached locally. Redis serves as this shared coordination layer. Without Redis, each RGW instance maintains an independent local cache, which reduces efficiency.

## Prerequisites

- Ceph cluster with RGW configured
- Redis 6.x or later installed and accessible from RGW hosts
- D3N local datacache enabled on RGW instances

## Installing Redis

```bash
# On RHEL/CentOS/Rocky
dnf install redis -y
systemctl enable --now redis

# On Ubuntu/Debian
apt-get install redis-server -y
systemctl enable --now redis-server

# Verify Redis is running
redis-cli ping
```

## Configuring Redis for D3N

Edit the Redis configuration for production use:

```bash
# /etc/redis/redis.conf
bind 0.0.0.0
port 6379
maxmemory 4gb
maxmemory-policy allkeys-lru
save ""
appendonly no
```

Restart Redis after changes:

```bash
systemctl restart redis
```

## Configuring RGW to Use Redis Backend

Set the D3N Redis configuration via Ceph config:

```bash
# Enable D3N with Redis backend
ceph config set client.rgw.myzone d3n_l1_local_datacache_enabled true
ceph config set client.rgw.myzone d3n_l1_datacache_persistent_path /var/lib/ceph/rgw/cache
ceph config set client.rgw.myzone d3n_l1_datacache_size 10737418240

# Configure Redis connection
ceph config set client.rgw.myzone rgw_d3n_l1_datacache_redis_url "redis://192.168.1.100:6379"
```

Or in `ceph.conf`:

```ini
[client.rgw.myzone]
d3n_l1_local_datacache_enabled = true
d3n_l1_datacache_persistent_path = /var/lib/ceph/rgw/cache
d3n_l1_datacache_size = 10737418240
rgw_d3n_l1_datacache_redis_url = redis://192.168.1.100:6379
```

## Using Redis Sentinel for High Availability

For production, use Redis Sentinel to avoid a single point of failure:

```bash
# Sentinel configuration - /etc/redis/sentinel.conf
sentinel monitor mymaster 192.168.1.100 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000

# Point RGW to Sentinel
ceph config set client.rgw.myzone rgw_d3n_l1_datacache_redis_url "redis://192.168.1.100:26379"
```

## Verifying Redis Coordination

```bash
# Monitor D3N keys in Redis
redis-cli -h 192.168.1.100 KEYS "d3n*" | head -20

# Check Redis memory usage
redis-cli -h 192.168.1.100 INFO memory | grep used_memory_human

# Watch Redis operations in real time
redis-cli -h 192.168.1.100 MONITOR | grep d3n
```

## Restart RGW After Configuration

```bash
# Rook-managed RGW - delete pods to trigger restart
kubectl -n rook-ceph delete pod -l app=rook-ceph-rgw

# Systemd-managed RGW
systemctl restart ceph-radosgw@rgw.myzone
```

## Summary

Configuring Redis as the D3N backend enables multiple RGW instances to share cache state, preventing redundant fetches from the backend cluster. Use Redis Sentinel for high availability, set an appropriate maxmemory policy, and verify coordination via Redis monitoring commands. This setup significantly improves cache hit rates in multi-RGW deployments.
