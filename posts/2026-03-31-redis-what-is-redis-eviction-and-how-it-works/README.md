# What Is Redis Eviction and How It Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Eviction, Memory

Description: A complete explanation of Redis eviction - how the eviction algorithm works, what policies are available, and how to choose the right one for your use case.

---

Redis eviction is the process of automatically removing keys when memory usage reaches the `maxmemory` limit. Understanding how it works helps you configure Redis appropriately for caching vs. persistent storage use cases.

## What Triggers Eviction

When a write command arrives and Redis is at or above `maxmemory`, Redis tries to evict keys before executing the write. If no keys can be evicted (e.g., with `noeviction` policy), Redis returns an error:

```text
(error) OOM command not allowed when used memory > 'maxmemory'.
```

Configure maxmemory:

```bash
redis-cli CONFIG SET maxmemory 2gb
redis-cli INFO memory | grep maxmemory_human
```

## Available Eviction Policies

```bash
redis-cli CONFIG GET maxmemory-policy
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

The available policies are:

- `noeviction` - return errors when memory is full (default)
- `allkeys-lru` - evict any key using Least Recently Used
- `volatile-lru` - evict keys with TTLs using LRU
- `allkeys-lfu` - evict any key using Least Frequently Used
- `volatile-lfu` - evict keys with TTLs using LFU
- `allkeys-random` - evict random keys
- `volatile-random` - evict random keys with TTLs
- `volatile-ttl` - evict keys with shortest remaining TTL

## How LRU Works in Redis

Redis does not implement a full LRU list. Instead it uses an approximate LRU algorithm: it samples a set of random keys and evicts the one with the oldest access time. The sample size is configurable:

```bash
redis-cli CONFIG GET maxmemory-samples
# Default: 5 - higher values are more accurate but use more CPU
redis-cli CONFIG SET maxmemory-samples 10
```

## How LFU Works in Redis

LFU (Least Frequently Used) tracks how often each key is accessed. Redis uses a probabilistic counter that decays over time, so recently accessed keys are preferred over old but once-popular keys:

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
redis-cli OBJECT FREQ mykey  # Check LFU counter
```

Configure LFU decay:

```bash
redis-cli CONFIG GET lfu-decay-time     # Default: 1 minute
redis-cli CONFIG GET lfu-log-factor     # Default: 10
```

## Monitoring Evictions

Track how many keys Redis has evicted:

```bash
redis-cli INFO stats | grep evicted_keys
```

High eviction rates indicate your cache is under-provisioned or your data doesn't fit the cache. Either increase `maxmemory` or reduce data stored in Redis.

## Choosing the Right Policy

Use case guidelines:
- Pure cache (all keys equally disposable): `allkeys-lru` or `allkeys-lfu`
- Mix of persistent and cached data: `volatile-lru` (set TTLs on cache keys only)
- Session store where all sessions should expire naturally: `volatile-ttl`
- Data store (no eviction acceptable): `noeviction`

## Summary

Redis eviction automatically removes keys when `maxmemory` is reached. The eviction policy controls which keys are removed. LRU and LFU are the most commonly used policies; LFU works better for skewed access patterns. Monitor evicted_keys to detect when your Redis instance is under memory pressure.
