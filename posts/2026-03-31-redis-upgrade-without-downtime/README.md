# How to Upgrade Redis Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Upgrade, Zero Downtime, Replication, Sentinel, High Availability

Description: Upgrade Redis to a newer version without downtime using replica promotion, rolling upgrades in Sentinel setups, and cluster rolling upgrades.

---

## Upgrade Strategies Overview

The correct strategy depends on your setup:
- **Standalone** - requires a brief maintenance window unless you use replica promotion
- **Sentinel** - rolling upgrade with replica-first approach
- **Cluster** - rolling upgrade one node at a time

## Pre-Upgrade Checklist

```bash
# Check current version
redis-server --version
redis-cli INFO server | grep redis_version

# Review release notes for breaking changes
# https://raw.githubusercontent.com/redis/redis/7.2/00-RELEASENOTES

# Test upgrade in staging first
# Backup RDB/AOF before upgrading
redis-cli BGSAVE
redis-cli BGREWRITEAOF
```

## Upgrading a Sentinel Setup (Zero Downtime)

For a setup with one primary and two replicas under Sentinel:

### Step 1 - Upgrade Replicas First

```bash
# On replica 1 - stop Redis
sudo systemctl stop redis

# Upgrade Redis package
sudo dnf install -y redis-7.2   # or your package manager

# Start Redis (it reconnects as replica automatically)
sudo systemctl start redis

# Verify replica is synced
redis-cli -p 6380 INFO replication | grep -E "role|master_link_status"
```

### Step 2 - Trigger Failover to Upgraded Replica

```bash
# Force Sentinel to elect the upgraded replica as the new primary
redis-cli -p 26379 SENTINEL FAILOVER mymaster

# Confirm the new primary is running the new version
redis-cli -p 6380 INFO server | grep redis_version
```

### Step 3 - Upgrade the Old Primary (Now a Replica)

```bash
# The old primary is now a replica - upgrade it safely
sudo systemctl stop redis
sudo dnf install -y redis-7.2
sudo systemctl start redis
redis-cli INFO replication | grep role
```

### Step 4 - Upgrade Sentinel Nodes

```bash
# Upgrade each Sentinel after all Redis nodes are updated
for sentinel_port in 26379 26380 26381; do
    redis-cli -p $sentinel_port INFO server | grep redis_version
    sudo systemctl stop redis-sentinel-$sentinel_port
    # Upgrade binary
    sudo systemctl start redis-sentinel-$sentinel_port
done
```

## Upgrading Redis Cluster (Rolling Upgrade)

Upgrade one node at a time, always starting with replicas:

```bash
#!/bin/bash
# Get all replica node addresses
redis-cli -p 7000 CLUSTER NODES | grep slave | awk '{print $2}' | while read node; do
    host=$(echo $node | cut -d: -f1)
    port=$(echo $node | cut -d: -f2 | cut -d@ -f1)
    echo "Upgrading replica $host:$port"

    # Stop the replica
    redis-cli -h $host -p $port SHUTDOWN SAVE

    # Upgrade (example for yum-managed install)
    ssh $host "sudo dnf install -y redis-7.2 && sudo systemctl start redis"

    # Wait for it to sync
    sleep 5
    redis-cli -h $host -p $port INFO replication | grep master_link_status
done
```

Then upgrade primaries one at a time using manual failover:

```bash
# Failover each primary to its replica before upgrading
redis-cli -h primary-host -p 7000 CLUSTER FAILOVER

# Wait for failover to complete
redis-cli -p 7000 CLUSTER INFO | grep cluster_state

# Upgrade the old primary (now a replica)
ssh primary-host "sudo dnf install -y redis-7.2 && sudo systemctl restart redis"
```

## Upgrading a Standalone Redis (Minimal Downtime)

If no replication is available, perform an in-place upgrade:

```bash
# 1. Save current data
redis-cli BGSAVE
redis-cli LASTSAVE

# 2. Stop Redis
sudo systemctl stop redis

# 3. Upgrade
sudo dnf install -y redis-7.2

# 4. Start Redis (loads existing RDB)
sudo systemctl start redis

# 5. Verify
redis-cli PING
redis-cli INFO server | grep redis_version
redis-cli DBSIZE
```

## Verifying After Upgrade

```bash
# Check version
redis-cli INFO server | grep redis_version

# Check replication is healthy
redis-cli INFO replication

# Run a basic command set
redis-cli SET post_upgrade_test "ok"
redis-cli GET post_upgrade_test

# Check for deprecation warnings in logs
sudo journalctl -u redis --since "5 minutes ago" | grep -i warn
```

## Rolling Back if Needed

Redis RDB files are generally backward-compatible but not always forward-compatible. To rollback:

```bash
# Stop new version
sudo systemctl stop redis

# Reinstall old version
sudo dnf install -y redis-7.0.15

# Restore from pre-upgrade RDB backup
cp /backup/dump.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# Start old version
sudo systemctl start redis
```

## Summary

Zero-downtime Redis upgrades require a replica-first rolling approach: upgrade replicas first, trigger a controlled failover so an upgraded replica becomes the new primary, then upgrade the remaining nodes. Redis Cluster supports the same strategy on a per-shard basis. Always test the upgrade on a staging replica before touching production primaries.
