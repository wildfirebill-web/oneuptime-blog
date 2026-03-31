# Redis Persistence Strategy Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, RDB, AOF, Durability, Best Practices

Description: A practical guide to choosing and configuring the right Redis persistence strategy based on your durability requirements and performance constraints.

---

## Redis Persistence Options

Redis offers three persistence approaches:

- **RDB (Redis Database)** - Periodic point-in-time snapshots
- **AOF (Append Only File)** - Log of every write command
- **RDB + AOF Hybrid** - Combined approach for fast load and high durability

Each has different tradeoffs between performance, recovery speed, and data loss risk.

## RDB Snapshotting

RDB forks the Redis process and writes a binary snapshot to disk. It is fast to load on restart.

```bash
# Configure RDB in redis.conf
# save <seconds> <changes>
save 900 1       # Save if 1+ keys changed in 900 seconds (15 min)
save 300 10      # Save if 10+ keys changed in 300 seconds (5 min)
save 60 10000    # Save if 10000+ keys changed in 60 seconds

# Disable RDB entirely
save ""

# Snapshot file name and location
dbfilename dump.rdb
dir /var/lib/redis
```

### RDB Tradeoffs

```text
Pros:
- Compact binary format - smaller files than AOF
- Fast restart - load entire dataset quickly
- Minimal performance impact (child process does the work)
- Good for backups and disaster recovery

Cons:
- Data loss between snapshots (up to 15 minutes with default config)
- Fork can cause brief latency spikes on large datasets
- Not suitable for near-zero RPO requirements
```

## AOF Logging

AOF logs every write command. On restart, Redis replays the log to rebuild the dataset.

```bash
# Enable AOF in redis.conf
appendonly yes
appendfilename "appendonly.aof"
dir /var/lib/redis

# fsync policy:
# always   - fsync after every write (safest, slowest)
# everysec - fsync every second (good balance) - DEFAULT
# no       - OS decides when to fsync (fastest, least safe)
appendfsync everysec
```

### AOF fsync Policy Comparison

```text
Policy      Data Loss Risk     Write Performance
------      --------------     -----------------
always      < 1 write          Slowest (~1/10 throughput)
everysec    < 1 second         Good (minor impact)
no          OS-dependent       Fastest (no Redis overhead)
```

### AOF Rewrite

AOF files grow over time. Redis can rewrite (compact) the AOF:

```bash
# Trigger manual AOF rewrite
redis-cli BGREWRITEAOF

# Auto-rewrite config in redis.conf
auto-aof-rewrite-percentage 100   # Rewrite when AOF is 100% bigger than base
auto-aof-rewrite-min-size 64mb   # Don't rewrite if AOF is smaller than 64MB

# Reduce fsync during rewrite to avoid I/O contention
no-appendfsync-on-rewrite yes
```

## RDB + AOF Hybrid Mode (Recommended)

Hybrid mode combines RDB's fast load time with AOF's durability:

```bash
# Enable hybrid mode in redis.conf
appendonly yes
aof-use-rdb-preamble yes   # AOF starts with RDB snapshot, then appends commands
appendfsync everysec
```

On restart, Redis loads the RDB preamble quickly, then replays only the commands added after the snapshot. This is the best choice for most production workloads.

## Choosing the Right Strategy

```text
Use Case                        Strategy                    Max Data Loss
--------                        --------                    -------------
Pure cache (rebuildable)        No persistence              All data
Loose durability OK             RDB only (default)          Up to 15 min
Good durability needed          AOF everysec                < 1 second
Near-zero data loss             AOF always                  < 1 write
Best balance (recommended)      RDB + AOF hybrid            < 1 second
```

## Backup Strategy

```bash
# Trigger a manual RDB snapshot
redis-cli BGSAVE

# Check when last save completed
redis-cli LASTSAVE

# Copy RDB file for backup (while Redis is running)
cp /var/lib/redis/dump.rdb /backups/redis/dump-$(date +%Y%m%d-%H%M%S).rdb

# Automate with cron - backup every hour
0 * * * * cp /var/lib/redis/dump.rdb /backups/redis/dump-$(date +\%Y\%m\%d-\%H).rdb
```

## Monitoring Persistence Health

```bash
# Check persistence status
redis-cli INFO persistence

# Key fields:
# rdb_last_save_time          - Unix timestamp of last RDB save
# rdb_last_bgsave_status      - ok or err
# aof_enabled                 - 0 or 1
# aof_rewrite_in_progress     - 0 or 1
# aof_last_write_status       - ok or err

# Check AOF file size
redis-cli INFO persistence | grep aof_current_size
```

## Disk Space Planning

```bash
# RDB file size is roughly the size of your dataset in memory
# AOF is typically 2-3x the dataset size (before rewrite)

# Check current file sizes
ls -lh /var/lib/redis/

# Monitor disk usage
df -h /var/lib/redis
```

## Disaster Recovery Testing

```bash
# Test RDB recovery
# 1. Stop Redis
systemctl stop redis

# 2. Corrupt or remove current RDB to simulate failure
mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.bak

# 3. Restore from backup
cp /backups/redis/dump-backup.rdb /var/lib/redis/dump.rdb

# 4. Start Redis and verify
systemctl start redis
redis-cli DBSIZE
```

## Summary

For most production workloads, RDB + AOF hybrid mode with appendfsync everysec is the recommended strategy - it provides fast restarts from the RDB snapshot and at most one second of data loss from AOF. Pure caches can safely disable persistence. High-durability workloads like financial data should use AOF always. Always test your recovery procedure before you need it.
