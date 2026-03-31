# How to Use redis-cli for Database Administration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, redis-cli, Administration, Database Management, CLI, DevOps

Description: Master redis-cli for daily database administration tasks including key inspection, memory analysis, configuration changes, replication monitoring, and cluster management.

---

## Connecting with redis-cli

```bash
# Basic connection
redis-cli

# Connect to a specific host and port
redis-cli -h 192.168.1.100 -p 6379

# Authenticate
redis-cli -h 192.168.1.100 -p 6379 -a your_password

# Use a specific database
redis-cli -n 3

# Run a single command without interactive mode
redis-cli GET mykey
```

## Key Inspection

```bash
# Check if key exists
EXISTS user:101

# Get the type of a key
TYPE user:101

# Get TTL in seconds (-1 = no expiry, -2 = key doesn't exist)
TTL user:101

# Get TTL in milliseconds
PTTL user:101

# Scan for keys matching a pattern (never use KEYS in production)
SCAN 0 MATCH "user:*" COUNT 100

# Continue scan with cursor
SCAN 150 MATCH "user:*" COUNT 100
```

## Memory Analysis

```bash
# Get memory used by a specific key
MEMORY USAGE user:101

# Analyze keys with largest memory footprint
redis-cli --bigkeys

# Get memory usage report
INFO memory

# Get per-key memory in bytes
MEMORY USAGE session:tok123 SAMPLES 5
```

## INFO Command for Server Health

```bash
# Full info
INFO

# Just replication info
INFO replication

# Just memory info
INFO memory

# Just persistence info
INFO persistence

# Stats info
INFO stats

# Clients info
INFO clients
```

## Configuration Management

```bash
# Get a config parameter
CONFIG GET maxmemory
CONFIG GET save
CONFIG GET bind

# Set a config parameter at runtime (no restart needed)
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru

# Persist current in-memory config to redis.conf
CONFIG REWRITE

# Reset statistics
CONFIG RESETSTAT
```

## Monitoring Active Commands

```bash
# Monitor all commands in real-time (use with caution in production)
MONITOR

# List slow queries (commands over slow-log-slower-than threshold)
SLOWLOG GET 25

# Reset slow log
SLOWLOG RESET

# Get count of slow log entries
SLOWLOG LEN
```

## Replication Administration

```bash
# Check replication status
INFO replication

# Make a node a replica of another
REPLICAOF 192.168.1.10 6379

# Promote replica to primary
REPLICAOF NO ONE

# Trigger a failover on a replica (Cluster mode)
CLUSTER FAILOVER
```

## Key Expiry Management

```bash
# Set TTL on existing key
EXPIRE user:101 3600

# Set TTL in milliseconds
PEXPIRE user:101 3600000

# Set absolute expiry timestamp (Unix seconds)
EXPIREAT user:101 1712000000

# Remove expiry (make permanent)
PERSIST user:101

# Check expiry
TTL user:101
```

## Bulk Key Operations with redis-cli Pipeline Mode

```bash
# Pipe mode for bulk inserts
cat commands.txt | redis-cli --pipe

# commands.txt example
echo -e "SET key1 val1\nSET key2 val2\nSET key3 val3" | redis-cli --pipe
```

## Database Management

```bash
# Count all keys in current database
DBSIZE

# Count keys in a specific DB
redis-cli -n 2 DBSIZE

# Flush current database
FLUSHDB

# Flush all databases (destructive - use with extreme caution)
FLUSHALL ASYNC

# Move a key to another database
MOVE user:101 2
```

## Debug and Diagnostics

```bash
# Check latency
redis-cli --latency

# Intrinsic server latency
redis-cli --intrinsic-latency 10

# Check for large keys
redis-cli --bigkeys

# Check hot keys (requires maxmemory-policy volatile-lfu or allkeys-lfu)
redis-cli --hotkeys

# Real-time stats display
redis-cli --stat

# Cluster node information
CLUSTER INFO
CLUSTER NODES
```

## Backup via redis-cli

```bash
# Trigger a background save
BGSAVE

# Check last save timestamp
LASTSAVE

# Download RDB from remote server
redis-cli -h remote-host --rdb /tmp/backup.rdb
```

## Summary

`redis-cli` is the primary tool for Redis database administration, covering key inspection, memory profiling, configuration management, replication monitoring, and bulk operations. Use SCAN instead of KEYS for safe key enumeration in production, CONFIG SET for runtime configuration changes, and `--bigkeys` and `--latency` flags for performance diagnostics.
