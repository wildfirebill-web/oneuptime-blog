# How Redis LRU and LFU Eviction Algorithms Work Internally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, LRU, LFU, Eviction, Memory Management, Internals

Description: Explore how Redis implements approximate LRU and LFU eviction using clock sampling and frequency counters to manage memory without a true LRU linked list.

---

## Why Redis Uses Approximate Eviction

True LRU eviction requires maintaining a doubly-linked list of all keys ordered by last access time, which adds 16 bytes of overhead per key and requires pointer updates on every read. For a 10 million key database, this adds 160 MB of overhead and significant CPU cost.

Redis instead implements **approximate LRU** and **approximate LFU** using small samples. This sacrifices perfect accuracy for dramatically lower memory and CPU overhead.

## The LRU Clock

Each `redisObject` (the internal wrapper for all Redis values) stores a 24-bit LRU clock value:

```text
redisObject {
  type:   4 bits   (STRING, LIST, HASH, etc.)
  encoding: 4 bits
  lru:   24 bits   <- LRU clock timestamp
  refcount: 32 bits
  ptr:    64 bits
}
```

The 24-bit clock stores seconds with a resolution of 1 second and wraps around every 194 days (`2^24 / 3600 / 24 = 194`). The global clock is updated by the server cron every 1000ms/`hz`.

When a key is accessed, its `lru` field is updated to the current clock value:

```c
robj->lru = LRU_CLOCK();  // called on every read/write
```

## Approximate LRU: Pool Sampling

When eviction is needed, Redis does **not** scan all keys. Instead it:

1. Samples `maxmemory-samples` keys at random (default 5)
2. Compares their LRU clock values
3. Evicts the key with the oldest LRU timestamp

```bash
redis-cli CONFIG GET maxmemory-samples
```

```text
maxmemory-samples: 5
```

Increase the sample size for better accuracy at higher CPU cost:

```bash
redis-cli CONFIG SET maxmemory-samples 10
```

With `maxmemory-samples 10`, the eviction quality approaches true LRU at roughly double the CPU cost.

### The Eviction Pool

Redis 3.0 added an **eviction pool** of 16 candidates to improve LRU accuracy. The pool retains the best candidates from previous sampling rounds, so each eviction benefits from accumulated observations rather than a single sample:

```text
Round 1: Sample 5 keys -> add best 16 candidates to pool
Round 2: Sample 5 keys -> merge with pool, keep best 16
...
Evict: Remove the worst candidate from the pool
```

This significantly improves approximation quality without increasing per-eviction cost.

## LFU: Frequency Counters

LFU (Least Frequently Used) eviction, introduced in Redis 4.0, tracks access frequency rather than recency. The `lru` field in `redisObject` is reused to store LFU data:

```text
LFU clock (16 bits) | LFU counter (8 bits)
```

- The 8-bit counter stores access frequency using a logarithmic approximation
- The 16-bit clock stores the last decrement time (in minutes)

### Logarithmic Counter

The 8-bit counter uses a probabilistic increment to approximate logarithmic counting:

```text
p = 1.0 / (counter * lfu_log_factor + 1)
if random() < p:
    counter++
```

With `lfu-log-factor 10` (default), the counter saturates at around 1 million accesses. This means:

| Counter Value | Approximate Accesses |
|---------------|---------------------|
| 0             | 0                   |
| 10            | ~100                |
| 20            | ~10,000             |
| 30            | ~1,000,000          |
| 255           | > 1 billion         |

### Counter Decay

To handle keys that were once hot but are now cold, the LFU counter decays over time based on the `lfu-decay-time` setting:

```bash
redis-cli CONFIG GET lfu-decay-time
```

```text
lfu-decay-time: 1
```

Every `lfu-decay-time` minutes of idle time, the counter is decremented by 1. This lets previously popular keys fall in rank after they stop being accessed.

```text
Key last accessed: 10 minutes ago
lfu-decay-time: 1
Counter decrease: 10 points
```

## Inspecting LFU Frequency

Use `OBJECT FREQ` to see the current LFU counter for a key:

```bash
redis-cli SET hotkey "value"
redis-cli OBJECT FREQ hotkey
```

```text
(integer) 4
```

Access the key many times and the counter increases:

```bash
for i in $(seq 1 1000); do redis-cli GET hotkey > /dev/null; done
redis-cli OBJECT FREQ hotkey
```

```text
(integer) 22
```

## Choosing Between LRU and LFU

### Use allkeys-lru or volatile-lru when:
- Access patterns are recency-based (recently accessed = likely needed soon)
- You have a streaming or sliding window workload
- Cache entries have roughly equal importance

### Use allkeys-lfu or volatile-lfu when:
- Some keys are much more popular than others (Pareto distribution)
- You want to retain "eternal hot keys" like frequently-accessed configuration or product catalog entries
- Access patterns are skewed (20% of keys get 80% of traffic)

## Eviction Policy Configuration

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lru
# or
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
```

In `redis.conf`:

```text
maxmemory 4gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10
lfu-log-factor 10
lfu-decay-time 1
```

## Monitoring Eviction Activity

```bash
redis-cli INFO stats | grep evicted
```

```text
evicted_keys:125000
```

High `evicted_keys` with low hit rate indicates your cache is too small or the eviction policy is suboptimal. Check the hit ratio:

```bash
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
```

```text
keyspace_hits:8500000
keyspace_misses:450000
```

```text
hit_rate = 8500000 / (8500000 + 450000) = 94.97%
```

## Summary

Redis implements approximate LRU using a 24-bit clock stored in each object and pool-based sampling to avoid maintaining a full sorted list. LFU uses a logarithmic counter with time-based decay stored in the same field. Both algorithms trade perfect accuracy for dramatic reductions in memory and CPU overhead. Understanding the sample size, pool behavior, and decay parameters lets you tune eviction quality and choose the right policy for your workload's access pattern.
