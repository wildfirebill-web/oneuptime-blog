# How to Choose Between RDB and AOF in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, RDB, AOF, Configuration

Description: A practical guide to choosing between Redis RDB snapshots and AOF persistence based on your durability, performance, and recovery requirements.

---

## Overview of RDB and AOF

Redis offers two primary persistence mechanisms:

- **RDB (Redis Database)** - Creates point-in-time snapshots of the dataset at configured intervals
- **AOF (Append-Only File)** - Logs every write command, allowing reconstruction of the dataset

You can use either or both simultaneously. Understanding the trade-offs will help you make the right choice.

## How RDB Works

RDB persistence creates binary snapshots of your dataset at defined intervals. Redis forks a child process to write the snapshot without blocking the main process.

Configuration example:

```text
# Save if at least 1 key changed in 900 seconds
save 900 1
# Save if at least 10 keys changed in 300 seconds
save 300 10
# Save if at least 10000 keys changed in 60 seconds
save 60 10000

dbfilename dump.rdb
dir /var/lib/redis
```

## How AOF Works

AOF appends every write command to a file. On restart, Redis replays all commands to rebuild the dataset.

Configuration example:

```text
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
```

## Comparing Durability

```text
Mechanism   | Max Data Loss         | Notes
------------|---------------------- |---------------------------------
RDB         | Minutes to hours      | Depends on save interval
AOF everysec| Up to ~1 second       | Background fsync thread
AOF always  | Zero (near)           | Significant performance cost
AOF no      | Depends on OS buffer  | Fastest but least safe
```

Use AOF `always` only if you absolutely cannot tolerate data loss and can accept the throughput reduction.

## Comparing Performance

RDB has lower overhead during normal operation - the fork happens occasionally and writes are done by a child process. AOF with `everysec` adds a small overhead per second but is generally acceptable.

Benchmark comparison (approximate):

```bash
# Test RDB-only throughput
redis-benchmark -n 100000 -q

# Test AOF always throughput (expect ~50-80% reduction)
redis-benchmark -n 100000 -q
```

AOF with `always` can reduce throughput by 50-80% depending on disk speed, so use SSD storage if choosing this option.

## Comparing Recovery Time

RDB loads faster because it is a binary format - Redis simply reads the snapshot directly:

```text
* Loading RDB produced by version 7.0.0
* RDB age 3 seconds
* RDB memory usage when created: 1.23G
* DB loaded from disk: 0.823 seconds
```

AOF recovery requires replaying all commands, which is slower:

```text
* DB loaded from append only file: 12.456 seconds
```

Redis 7.0's multi-part AOF reduces this by using an RDB base file with incremental AOF.

## Comparing File Size

RDB files are compact binary snapshots. AOF files grow with every write and require periodic compaction via `BGREWRITEAOF`. After rewrite, AOF and RDB sizes are comparable.

Check sizes:

```bash
ls -lh /var/lib/redis/dump.rdb
ls -lh /var/lib/redis/appendonly.aof
```

## When to Use RDB Only

Choose RDB-only when:

- You can tolerate some data loss (cache use cases)
- You need the fastest restart times
- You are doing periodic backups and snapshots fit your backup workflow
- Your dataset is very large and AOF replay would be too slow

```text
# redis.conf for RDB-only
save 900 1
save 300 10
save 60 10000
appendonly no
```

## When to Use AOF Only

Choose AOF-only when:

- Data durability is critical and you cannot accept more than 1 second of data loss
- You need a complete audit trail of write operations
- Your dataset is smaller and replay time is acceptable

```text
# redis.conf for AOF-only
save ""
appendonly yes
appendfsync everysec
```

## When to Use Both

Using both provides maximum safety. RDB gives you fast restores and periodic backups; AOF gives you fine-grained durability:

```text
# redis.conf for both RDB and AOF
save 900 1
save 300 10
appendonly yes
appendfsync everysec
```

When both are enabled, Redis uses AOF on startup (it is more complete) and ignores the RDB file unless AOF is missing.

## Decision Guide

```text
Use case                        | Recommendation
--------------------------------|------------------------
Pure cache, loss is OK          | No persistence or RDB
Moderate durability needed      | RDB + AOF everysec
Strict durability, SSD server   | AOF always
Large dataset, fast recovery    | RDB-only or multi-part AOF
Audit trail required            | AOF
```

## Summary

RDB and AOF serve different needs: RDB is compact, fast to load, and sufficient when some data loss is acceptable, while AOF provides fine-grained durability at the cost of slower recovery and higher I/O. For most production deployments, combining both mechanisms - RDB for fast backups and AOF with `everysec` for durability - is the recommended approach. The right choice depends on your tolerance for data loss, your server I/O capacity, and your recovery time objectives.
