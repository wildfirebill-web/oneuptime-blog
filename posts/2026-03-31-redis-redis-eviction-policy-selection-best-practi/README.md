# Redis Eviction Policy Selection Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Eviction Policy, Memory Management, Configuration, Best Practice

Description: A practical guide to selecting the right Redis eviction policy for your use case, covering all eight policies with real-world decision criteria.

---

## What Is an Eviction Policy?

When Redis reaches its maxmemory limit, it needs to decide what to do when new data arrives. The eviction policy determines which keys (if any) to remove to make space.

```bash
# Check current eviction policy
redis-cli CONFIG GET maxmemory-policy

# Set an eviction policy
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

## The Eight Eviction Policies

```text
Policy              Scope       Algorithm   Description
------              -----       ---------   -----------
noeviction          All         None        Return error when full
allkeys-lru         All keys    LRU         Evict least recently used
volatile-lru        TTL keys    LRU         Evict LRU from keys with TTL
allkeys-lfu         All keys    LFU         Evict least frequently used
volatile-lfu        TTL keys    LFU         Evict LFU from keys with TTL
allkeys-random      All keys    Random      Evict random keys
volatile-random     TTL keys    Random      Evict random from TTL keys
volatile-ttl        TTL keys    TTL         Evict keys with soonest expiry
```

## Policy Deep Dives

### noeviction (Default)

Returns an error for write commands when memory is full. Data is never evicted.

```bash
# When memory is full, writes return:
# COMMAND: SET new-key value
# RESPONSE: (error) OOM command not allowed when used memory > 'maxmemory'
```

**Use when:** Redis is a primary data store (queues, session store) where data loss is unacceptable. Pair with proactive monitoring and alerting.

### allkeys-lru

Evicts the least recently used key from all keys, regardless of TTL.

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET maxmemory-samples 10  # Higher = more accurate LRU
```

**Use when:** Redis is a cache where all keys are equally valuable, and access recency predicts future value.

### volatile-lru

Same as allkeys-lru, but only evicts keys that have an expiry set.

**Use when:** You mix cached data (with TTL) and permanent data (without TTL) in the same Redis instance. Permanent keys are never evicted.

```javascript
// Permanent keys - no TTL, never evicted by volatile-lru
await redis.set('config:feature-flags', JSON.stringify(flags));

// Cached keys - have TTL, eligible for eviction
await redis.setex('cache:user:42:profile', 3600, JSON.stringify(profile));
```

### allkeys-lfu

Evicts the least frequently used key. LFU tracks how often each key is accessed.

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
```

**Use when:** Access frequency is a better predictor of future value than recency. A key accessed 1000 times is more valuable than one accessed once an hour ago.

### volatile-lfu

LFU eviction only from keys with TTL set.

**Use when:** Same reasoning as volatile-lru but you prefer frequency over recency as the eviction metric.

### allkeys-random

Evicts a random key from all keys.

**Use when:** Access patterns are perfectly uniform (rare in practice). Simpler than LRU/LFU and useful for benchmarking or when you know all keys are equally valuable.

### volatile-random

Evicts a random key from keys with TTL set.

**Use when:** Cached data has random access patterns and you don't want to track LRU/LFU overhead.

### volatile-ttl

Evicts keys with the shortest remaining TTL first.

**Use when:** You want to evict data that is about to expire anyway, minimizing the impact of eviction.

## Decision Tree

```text
Is Redis a primary data store (no data loss tolerable)?
  YES --> noeviction (with alerts at 80% capacity)

All keys are cache entries (all rebuildable from DB)?
  YES, access recency matters --> allkeys-lru
  YES, access frequency matters --> allkeys-lfu
  YES, uniform access --> allkeys-random

Mix of permanent and cached data?
  YES, recency matters --> volatile-lru
  YES, frequency matters --> volatile-lfu
  YES, evict-soon-expiring first --> volatile-ttl
```

## Configuring maxmemory-samples

LRU and LFU in Redis are approximations. The `maxmemory-samples` setting controls accuracy:

```bash
# Default is 5 - a good balance of speed and accuracy
redis-cli CONFIG SET maxmemory-samples 5

# Higher value (10) = more accurate, slightly more CPU
redis-cli CONFIG SET maxmemory-samples 10

# For most production cases, 10 is recommended
```

## Monitoring Evictions

```bash
# Check eviction stats
redis-cli INFO stats | grep evicted_keys

# Watch evictions in real time
watch -n 1 "redis-cli INFO stats | grep evicted_keys"

# Get eviction rate over time
redis-cli INFO all | grep evicted
```

## Changing Policy at Runtime

```bash
# Change without restart
redis-cli CONFIG SET maxmemory-policy volatile-lru

# Save to config file
redis-cli CONFIG REWRITE
```

## Complete Configuration Example

```text
# redis.conf for a cache server
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# redis.conf for a session store (no eviction)
maxmemory 2gb
maxmemory-policy noeviction
```

## Summary

Selecting the right eviction policy depends on your use case. For pure caches, allkeys-lru or allkeys-lfu are the best choices. For mixed workloads with permanent and temporary data, volatile-lru or volatile-lfu protect critical keys. For primary data stores, noeviction with proactive capacity monitoring is the safe default. Always monitor eviction counts after setting a policy to validate your choice.
