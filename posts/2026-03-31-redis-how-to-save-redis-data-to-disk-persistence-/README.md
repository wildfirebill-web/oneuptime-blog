# How to Save Redis Data to Disk (Persistence Basics)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, RDB, AOF, Beginner, Data Safety, Configuration

Description: A beginner-friendly guide to Redis persistence, explaining RDB snapshots and AOF logging so your data survives server restarts and crashes.

---

## Why Persistence Matters

Redis stores everything in memory. By default, if Redis restarts, all data is gone. Persistence means Redis saves a copy of your data to disk so it can reload it on startup.

## Two Ways to Save Data

Redis offers two persistence mechanisms:

```text
RDB (Redis Database Backup)
  - Saves a snapshot of all data at intervals
  - Like a full backup at a point in time
  - Fast to load, small file size
  - Risk: data written since last snapshot is lost on crash

AOF (Append Only File)
  - Logs every write command as it happens
  - Like a transaction log
  - More durable (less data loss risk)
  - Slightly larger files, a bit slower on restart
```

## Checking Your Current Persistence Settings

```bash
# Connect to Redis
redis-cli

# Check RDB settings
CONFIG GET save

# Check AOF settings
CONFIG GET appendonly
CONFIG GET appendfsync
```

## RDB Snapshots

RDB saves a binary snapshot of all your data. By default, Redis saves in these cases:

```bash
# Default save settings (from redis.conf)
save 3600 1      # Save if at least 1 key changed in last hour
save 300 100     # Save if at least 100 keys changed in last 5 minutes
save 60 10000    # Save if at least 10000 keys changed in last minute
```

### Trigger a Manual Save

```bash
# Background save (non-blocking - recommended)
redis-cli BGSAVE

# Foreground save (blocks Redis until done - avoid in production)
redis-cli SAVE

# Check when last save happened
redis-cli LASTSAVE
# Returns Unix timestamp of last successful RDB save
```

### RDB File Location

```bash
# Check where the RDB file is saved
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename

# Usually: /var/lib/redis/dump.rdb
ls -la /var/lib/redis/dump.rdb
```

### Configure RDB in redis.conf

```text
# Save every 15 minutes if at least 1 key changed
save 900 1

# Save every 5 minutes if at least 10 keys changed
save 300 10

# File name and location
dbfilename dump.rdb
dir /var/lib/redis
```

## AOF (Append Only File)

AOF records every write command. When Redis restarts, it replays these commands to rebuild the dataset.

### Enabling AOF

```bash
# Enable AOF at runtime (takes effect immediately)
redis-cli CONFIG SET appendonly yes

# Or in redis.conf
# appendonly yes
```

### AOF fsync Options

```text
appendfsync always    # Safest: sync to disk after every command
appendfsync everysec  # Good balance: sync once per second (default)
appendfsync no        # Fastest: let the OS decide (least safe)
```

```bash
# Check current setting
redis-cli CONFIG GET appendfsync

# Change to everysec (recommended)
redis-cli CONFIG SET appendfsync everysec
```

### AOF File Location

```bash
# Default AOF file
ls -la /var/lib/redis/appendonly.aof

# Check file size
redis-cli INFO persistence | grep aof_current_size
```

## Checking Persistence Status

```bash
# Full persistence info
redis-cli INFO persistence

# Key fields to check:
# rdb_last_bgsave_status    - ok or err
# rdb_last_save_time        - timestamp of last save
# aof_enabled               - 0 or 1
# aof_last_write_status     - ok or err
```

## Which Should You Use?

```text
Your situation                  Recommendation
--------------                  --------------
Just getting started            RDB (it's on by default)
Can't afford to lose any data   AOF with everysec
Cache only (data is rebuildable) No persistence needed
Best of both worlds             RDB + AOF together
```

## Enabling Both RDB and AOF (Recommended)

```bash
# Enable both persistence methods
redis-cli CONFIG SET save "900 1 300 10 60 10000"
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec

# Enable hybrid AOF (starts with RDB snapshot for fast load)
redis-cli CONFIG SET aof-use-rdb-preamble yes
```

## Disabling Persistence (Cache Mode)

If you're using Redis as a pure cache and can rebuild data from your database:

```bash
# Disable RDB
redis-cli CONFIG SET save ""

# Disable AOF
redis-cli CONFIG SET appendonly no
```

## Backing Up Your Data

```bash
# Create a backup of the RDB file
cp /var/lib/redis/dump.rdb /backups/dump-$(date +%Y%m%d).rdb

# Or trigger a save and copy
redis-cli BGSAVE
# Wait a moment, then copy the file
```

## What Happens on Redis Restart

```text
With RDB only:
  Redis loads dump.rdb -> all data from last snapshot is available

With AOF only:
  Redis replays appendonly.aof -> reconstructs all writes

With both RDB + AOF:
  Redis loads the AOF (which starts with RDB snapshot if hybrid mode enabled)
  -> fastest load + most recent data
```

## Summary

Redis persistence prevents data loss on restarts and crashes. RDB takes periodic snapshots - fast to load but risks losing recent writes. AOF logs every write command - more durable but slightly slower. For most applications, enable both with AOF set to everysec for a good balance of durability and performance. For pure caches where data can be rebuilt, persistence can safely be disabled to maximize performance.
