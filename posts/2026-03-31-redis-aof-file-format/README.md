# How Redis AOF File Format Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, AOF, Internal, Durability

Description: Learn the text-based structure of Redis AOF files, how commands are appended, how rewrites compact the log, and how to inspect and repair AOF files.

---

AOF (Append Only File) persistence records every write command Redis executes. On restart, Redis replays the AOF to reconstruct the dataset. The format is human-readable and built on the Redis Serialization Protocol (RESP).

## AOF File Structure

Each command in the AOF is stored in RESP format:

```text
*3\r\n          -- array of 3 elements
$3\r\n          -- bulk string of length 3
SET\r\n         -- command name
$5\r\n          -- bulk string of length 5
mykey\r\n       -- key
$7\r\n          -- bulk string of length 7
myvalue\r\n     -- value
```

Every write command is stored this way: `SET`, `HSET`, `LPUSH`, `EXPIRE`, etc.

## Viewing the AOF

```bash
tail -50 /var/lib/redis/appendonly.aof
```

Or count how many commands it holds:

```bash
grep -c "^\*" /var/lib/redis/appendonly.aof
```

## AOF Fsync Policies

The AOF offers three durability modes:

```text
# redis.conf
appendfsync always     # fsync after every command (safest, slowest)
appendfsync everysec   # fsync once per second (good balance)
appendfsync no         # let OS decide when to flush (fastest, risky)
```

Most production systems use `everysec`, risking at most one second of data loss on crash.

## AOF Rewrite

Over time the AOF grows large with redundant commands:

```text
SET counter 1
SET counter 2
SET counter 3
```

After rewrite, only the final state is kept:

```text
SET counter 3
```

Trigger a rewrite manually:

```bash
redis-cli BGREWRITEAOF
```

Or configure auto-rewrite:

```text
# redis.conf
auto-aof-rewrite-percentage 100  # rewrite when AOF doubles in size
auto-aof-rewrite-min-size 64mb
```

## Multi-Part AOF (Redis 7.0+)

Redis 7.0 introduced the multi-part AOF, splitting the log into:

```text
appendonlydir/
  base.rdb         -- base snapshot
  incr.aof.1       -- incremental log since base
  incr.aof.2       -- latest incremental log
  manifest         -- tracks which files make up the full AOF
```

This avoids blocking the main thread during rewrite by using a fresh incremental log while the rewrite runs in the background.

## Repairing a Corrupt AOF

If Redis crashes mid-write, the AOF tail may be incomplete:

```bash
redis-check-aof /var/lib/redis/appendonly.aof
# 0x00a1fb3: Expected \r\n, got: 69
# AOF analyzed: size=659923, ok_up_to=659897, diff=26

redis-check-aof --fix /var/lib/redis/appendonly.aof
# Successfully truncated AOF to last valid command
```

## Monitoring AOF Health

```bash
redis-cli INFO persistence | grep aof
# aof_enabled:1
# aof_rewrite_in_progress:0
# aof_last_rewrite_time_sec:4
# aof_current_size:1048576
# aof_base_size:524288
```

## Summary

The Redis AOF stores write commands in human-readable RESP format, fsyncing to disk based on your configured policy. Rewrites compact the log by replacing command sequences with their final state. Redis 7.0's multi-part AOF improves rewrite safety by separating base snapshots from incremental logs. Use `redis-check-aof --fix` to recover from partial writes after a crash.
