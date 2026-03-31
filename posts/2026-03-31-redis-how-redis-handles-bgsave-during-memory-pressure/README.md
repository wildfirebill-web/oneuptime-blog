# How Redis Handles BGSAVE During Memory Pressure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, BGSAVE, Memory

Description: Learn how Redis BGSAVE works under memory pressure - the fork mechanism, copy-on-write overhead, and how to reduce memory spikes during RDB snapshots.

---

BGSAVE creates a point-in-time RDB snapshot by forking a child process. Under memory pressure, this fork can cause the total memory usage to temporarily spike up to 2x, potentially triggering the OOM killer. Understanding how this works helps you plan capacity and tune behavior.

## How BGSAVE Uses Fork

When BGSAVE runs, Redis forks a child process. The child gets a copy of the parent's memory space via copy-on-write (COW). The child writes the RDB file while the parent continues serving requests.

```bash
redis-cli BGSAVE
redis-cli INFO persistence | grep -E "rdb_bgsave_in_progress|rdb_last_bgsave_time_sec"
```

## Copy-on-Write Memory Overhead

At the moment of fork, memory is shared between parent and child. But every page the parent writes to during the snapshot creates a copy - one for the parent, one for the child. Under high write load, this can nearly double memory usage temporarily.

Monitor COW overhead:

```bash
redis-cli INFO memory | grep "used_memory_rss_human"
```

`used_memory_rss` represents actual OS-level memory usage including COW pages. If `used_memory_rss` is much larger than `used_memory`, COW overhead is significant.

## Checking Fork Time

Fork itself can be slow when memory usage is high:

```bash
redis-cli INFO stats | grep "latest_fork_usec"
```

Fork time over 200ms indicates problems. Common causes:
- Large datasets
- Transparent huge pages (THP) enabled

Disable THP to reduce fork latency:

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

## Memory Pressure Warning Signs

Before BGSAVE, check available memory:

```bash
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"
```

Ensure there is enough headroom for COW overhead. Rule of thumb: free memory should be at least equal to the write-active working set during snapshot time.

## Failing BGSAVE

Redis fails BGSAVE if it cannot fork. Check the error:

```bash
redis-cli INFO persistence | grep "rdb_last_bgsave_status"
```

If BGSAVE fails, Redis may reject writes depending on `stop-writes-on-bgsave-error`:

```bash
redis-cli CONFIG GET stop-writes-on-bgsave-error
# Disable if you want writes to continue even after failed BGSAVE
redis-cli CONFIG SET stop-writes-on-bgsave-error no
```

## Scheduling BGSAVE During Off-Peak Hours

Disable automatic RDB saves and trigger BGSAVE manually during off-peak windows:

```bash
# redis.conf - disable automatic saves
save ""

# Trigger manually via cron
redis-cli BGSAVE
```

## Using WAIT to Confirm Replication Before BGSAVE

If using replication, ensure replicas are in sync before triggering BGSAVE:

```bash
redis-cli WAIT 1 5000  # Wait for 1 replica within 5 seconds
redis-cli BGSAVE
```

## Summary

BGSAVE under memory pressure is risky because fork + copy-on-write can temporarily double Redis memory usage. Monitoring fork time, disabling transparent huge pages, maintaining adequate memory headroom, and scheduling snapshots during low-write periods are the primary strategies for managing BGSAVE safely.
