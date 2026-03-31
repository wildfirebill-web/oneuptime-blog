# How to Use redis-shake for Redis Data Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, redis-shake, Migration, Data Transfer, DevOps

Description: Learn how to use redis-shake to migrate data between Redis instances, filter keys, transform data, and validate migration results in production.

---

redis-shake (also known as RedisShake) is an open-source tool from Alibaba for migrating data between Redis instances. It supports sync mode (live replication), restore mode (from RDB files), and scan mode (key-by-key copy), making it flexible for different migration scenarios.

## Installation

```bash
# Download the latest release
wget https://github.com/alibaba/RedisShake/releases/latest/download/redis-shake.tar.gz
tar xzf redis-shake.tar.gz
ls redis-shake/
# redis-shake  redis-shake.toml  (binary and sample config)
```

## Configuration File

redis-shake uses a TOML configuration file:

```toml
# redis-shake.toml

[source]
type = "standalone"          # standalone, cluster, sentinel
address = "source-host:6379"
username = ""
password = "source-password"
tls = false

[target]
type = "standalone"
address = "target-host:6379"
username = ""
password = "target-password"
tls = false

[advanced]
# Number of concurrent goroutines
parallel = 4

# Key filter - only migrate keys matching pattern
# filter_key_pattern = "session:*"

# Log level: debug, info, warning, error
log_level = "info"
log_file = "redis-shake.log"
```

## Sync Mode (Live Replication)

Sync mode is the most powerful: it does an initial full sync then streams live changes, making it suitable for near-zero downtime migration.

```bash
# Run sync mode
./redis-shake redis-shake.toml

# redis-shake output will show:
# [INFO] source: standalone standalone-source-host:6379
# [INFO] target: standalone standalone-target-host:6379
# [INFO] syncing...
# [INFO] all entries synced, total: 124532
# [INFO] start incremental sync
```

While in incremental sync, you can monitor progress:

```bash
# Check key count on target
redis-cli -h target-host -a "target-password" DBSIZE

# Monitor redis-shake log
tail -f redis-shake.log
```

## Restore Mode (From RDB File)

If you have an RDB dump file, use restore mode:

```toml
[source]
type = "rdb"
address = "/path/to/dump.rdb"

[target]
type = "standalone"
address = "target-host:6379"
password = "target-password"
```

```bash
./redis-shake redis-shake.toml
```

## Scan Mode (Key-by-Key Copy)

Scan mode reads keys from the source using SCAN and writes them to the target. It is slower but works when you cannot use replication:

```toml
[source]
type = "scan"
address = "source-host:6379"
password = "source-password"

[target]
type = "standalone"
address = "target-host:6379"
password = "target-password"
```

## Filtering Keys During Migration

```toml
[advanced]
# Migrate only keys matching a pattern
filter_key_pattern = "user:*"

# Exclude keys matching a pattern
exclude_key_pattern = "temp:*"

# Migrate only specific database number
source_db = 0
target_db = 0
```

## Cluster to Cluster Migration

```toml
[source]
type = "cluster"
address = "source-cluster-node1:6379"
password = "source-password"

[target]
type = "cluster"
address = "target-cluster-node1:6379"
password = "target-password"
```

## Validate Migration Results

```bash
# Compare key counts
SOURCE_KEYS=$(redis-cli -h source-host -a "source-password" DBSIZE)
TARGET_KEYS=$(redis-cli -h target-host -a "target-password" DBSIZE)
echo "Source: $SOURCE_KEYS, Target: $TARGET_KEYS"

# Spot check specific keys
redis-cli -h source-host -a "source-password" GET user:1001
redis-cli -h target-host -a "target-password" GET user:1001

# Compare TTLs
redis-cli -h source-host -a "source-password" TTL session:abc123
redis-cli -h target-host -a "target-password" TTL session:abc123
```

## Cutover Steps with redis-shake

1. Start redis-shake in sync mode
2. Wait for initial sync to complete (watch for "start incremental sync" in logs)
3. Monitor the offset gap - it should stay near zero
4. Update application connection strings to point to target
5. Stop redis-shake after confirming traffic is flowing to target

```bash
# Stop redis-shake gracefully
kill -SIGTERM $(pgrep redis-shake)
```

## Summary

redis-shake is the most capable tool for Redis migrations, supporting live sync, RDB restore, and key filtering. Use sync mode for production migrations where downtime must be minimal. Validate key counts and data integrity after migration before decommissioning the source instance.
