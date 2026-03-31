# How Redis Fork Works During BGSAVE and Its Memory Impact

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, BGSAVE, Performance, Internal

Description: Understand how Redis uses fork and copy-on-write during BGSAVE, why memory can spike during snapshots, and how to minimize the performance impact.

---

Every time Redis creates an RDB snapshot or rewrites the AOF file, it calls `fork()` to create a child process. The interaction between fork, copy-on-write, and write traffic determines how much extra memory is used and how much latency spikes during the snapshot.

## How Fork Works in Redis

When Redis runs `BGSAVE`:

1. `fork()` creates a child process that is an exact copy of the parent
2. On Linux, this is nearly instantaneous - both processes share the same physical memory pages
3. The child reads the shared pages and writes them to the RDB file
4. The parent continues serving client requests

The key property is copy-on-write (COW): the kernel only duplicates a memory page when either the parent or child writes to it. If no writes occur during the snapshot, no extra memory is needed.

## Measuring Fork Latency

The `fork()` system call itself takes time proportional to the number of virtual memory pages, not physical memory usage:

```bash
redis-cli INFO stats | grep latest_fork_usec
```

```text
latest_fork_usec:12500   # 12.5ms for this instance
```

Typical values:

- 1 GB dataset: ~1-3 ms
- 10 GB dataset: ~10-25 ms
- 100 GB dataset: ~100-250 ms

During this brief window, Redis cannot process any commands (fork is synchronous in the parent).

## Measuring Copy-on-Write Memory Overhead

Every page the parent writes during the save must be duplicated:

```bash
redis-cli INFO persistence | grep rdb_last_cow_size
```

```text
rdb_last_cow_size:52428800   # 50 MB of COW overhead
```

Calculate COW as a percentage of dataset size:

```bash
redis-cli INFO memory | grep used_memory_human
redis-cli INFO persistence | grep rdb_last_cow_size
```

For a 2 GB dataset with 50 MB COW, total memory during save is 2.05 GB. For a write-heavy workload with high COW, memory can temporarily approach 2x the dataset size.

## Why THP Makes This Worse

With Transparent Huge Pages enabled, each memory page is 2 MB instead of 4 KB. A single write to any byte in a 2 MB page triggers copying the entire 2 MB. This amplifies COW overhead by up to 512x:

```bash
# Before disabling THP
rdb_last_cow_size:524288000   # 500 MB

# After disabling THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# Next BGSAVE:
rdb_last_cow_size:8388608     # 8 MB
```

## Reducing Memory Impact During BGSAVE

### Strategy 1: Reduce Write Rate During Snapshots

Schedule BGSAVE during low-traffic periods:

```bash
# Trigger BGSAVE at 2 AM via cron
0 2 * * * redis-cli BGSAVE
```

### Strategy 2: Increase Memory Headroom

Never let Redis use more than 50-60% of available RAM when using RDB:

```text
# If the system has 16 GB RAM, set maxmemory to 8 GB
maxmemory 8gb
```

### Strategy 3: Monitor and Alert

Set up monitoring for COW and fork latency:

```bash
#!/bin/bash
FORK_US=$(redis-cli INFO stats | grep latest_fork_usec | cut -d: -f2 | tr -d '\r')
COW_BYTES=$(redis-cli INFO persistence | grep rdb_last_cow_size | cut -d: -f2 | tr -d '\r')

if [ "$FORK_US" -gt 100000 ]; then
    echo "ALERT: Fork took ${FORK_US}us"
fi

if [ "$COW_BYTES" -gt 1073741824 ]; then
    echo "ALERT: COW overhead exceeds 1 GB"
fi
```

## BGSAVE vs SAVE

Never use `SAVE` (synchronous save) in production:

```bash
# SAVE blocks Redis completely until done - DO NOT use in production
redis-cli SAVE

# BGSAVE forks a child and returns immediately
redis-cli BGSAVE
```

`SAVE` holds the global lock for the entire duration of the snapshot, blocking all client requests.

## Summary

Redis BGSAVE uses `fork()` followed by copy-on-write to create snapshots without stopping the server. Fork itself causes a brief latency spike proportional to dataset size, and subsequent writes during the snapshot consume additional memory via COW. Disable Transparent Huge Pages to reduce COW amplification, provision memory headroom of at least 1.5x your dataset size, and schedule BGSAVE during low-traffic windows to minimize impact.
