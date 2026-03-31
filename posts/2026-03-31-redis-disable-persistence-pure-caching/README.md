# How to Disable Persistence in Redis for Pure Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Performance, Configuration

Description: Disable Redis RDB and AOF persistence for pure caching use cases to maximize throughput, reduce memory usage, and eliminate disk I/O overhead.

---

When Redis is used exclusively as a cache (where data can be regenerated from a primary database on restart), persistence adds unnecessary overhead. Disabling RDB and AOF eliminates disk I/O, reduces memory usage from copy-on-write, and removes fork latency spikes.

## Disabling RDB Snapshots

The `save` directive with an empty string disables all automatic snapshots:

```text
# redis.conf
save ""
```

Or at runtime:

```bash
redis-cli CONFIG SET save ""
redis-cli CONFIG GET save
```

After disabling, verify no background save is scheduled:

```bash
redis-cli INFO persistence | grep rdb_last_bgsave_status
```

## Disabling AOF

```text
# redis.conf
appendonly no
```

At runtime:

```bash
redis-cli CONFIG SET appendonly no
```

When you disable AOF on a running instance that had it enabled, the AOF file is no longer written but still exists on disk until you remove it.

## Full No-Persistence Configuration

```text
# redis.conf - pure caching mode
save ""
appendonly no

# Optional: aggressively evict keys when memory is full
maxmemory 4gb
maxmemory-policy allkeys-lru

# Reduce copy-on-write overhead (no forks needed for saves)
# These remain important for replica syncs
```

## Ensuring stop-writes-on-bgsave-error Does Not Block Writes

If persistence is disabled, confirm that `stop-writes-on-bgsave-error` will not interfere:

```text
# redis.conf
stop-writes-on-bgsave-error no
```

With persistence disabled, there is no background save to fail, but this prevents edge cases:

```bash
redis-cli CONFIG SET stop-writes-on-bgsave-error no
```

## Removing Existing Persistence Files

After disabling persistence, clean up old files to reclaim disk space:

```bash
# Check current file locations
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename

# Remove after confirming Redis is running correctly without them
sudo rm /var/lib/redis/dump.rdb
sudo rm /var/lib/redis/appendonly.aof
```

## Verifying No Persistence Activity

```bash
redis-cli INFO persistence
```

```text
rdb_changes_since_last_save:0
rdb_last_save_time:1711900800
rdb_last_bgsave_status:ok
rdb_current_bgsave_time_sec:-1
aof_enabled:0
```

`aof_enabled:0` confirms AOF is off. `rdb_current_bgsave_time_sec:-1` means no save is running.

## Performance Gains

Measure throughput before and after disabling persistence:

```bash
# With persistence
redis-benchmark -n 1000000 -q

# After disabling
redis-cli CONFIG SET save ""
redis-cli CONFIG SET appendonly no
redis-benchmark -n 1000000 -q
```

Typical improvement: 15-30% higher throughput and more consistent latency (no periodic fork spikes).

## Handling Restarts

With persistence disabled, all data is lost on restart. Applications must handle a cold Redis cache:

```python
def get_user(user_id):
    key = f"user:{user_id}"
    cached = redis_client.get(key)

    if cached is None:
        # Cache miss - load from database
        user = db.query("SELECT * FROM users WHERE id = %s", user_id)
        redis_client.setex(key, 3600, serialize(user))
        return user

    return deserialize(cached)
```

Use TTLs on all cache keys so stale data is eventually evicted:

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

## Summary

Disable Redis persistence for pure caching by setting `save ""` and `appendonly no`. This eliminates disk I/O, fork latency spikes, and copy-on-write memory overhead. Set `maxmemory` with an LRU eviction policy to manage cache size, and ensure your application handles cache misses gracefully to recover from restarts.
