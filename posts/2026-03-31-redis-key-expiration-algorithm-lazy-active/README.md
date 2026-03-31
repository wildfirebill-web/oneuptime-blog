# How Redis Key Expiration Algorithm Works (Lazy + Active)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key Expiration, TTL, Internals, Memory Management

Description: Learn how Redis combines lazy and active expiration strategies to reclaim memory from expired keys without blocking the event loop.

---

## Overview of Redis Expiration

When you set a TTL on a key, Redis records the absolute expiration timestamp in a separate hash table called the `expires` dictionary. Redis does not immediately delete expired keys - instead it uses two complementary strategies to reclaim them: **lazy expiration** and **active expiration**.

```bash
redis-cli SET session:abc "data" EX 3600
redis-cli TTL session:abc
```

```text
(integer) 3599
```

Internally, Redis stores:

```text
expires dict:
  "session:abc" -> 1711903600000 (Unix ms timestamp)
```

## Lazy Expiration

Lazy expiration checks whether a key has expired at the moment it is accessed. When any command reads or modifies a key, Redis calls `expireIfNeeded()` first:

1. Look up the key in the `expires` dict
2. If the key has an expiry and the current time exceeds it, delete the key
3. Return NULL to the caller as if the key never existed

```text
GET expired_key
  -> expireIfNeeded("expired_key")
     -> expires["expired_key"] = 1711900000
     -> now = 1711910000
     -> 1711910000 > 1711900000 -> DELETE key
  -> return NULL
```

This is efficient because no background scan is needed - expired keys are cleaned up as they are accessed. However, keys that are never read again will linger in memory indefinitely.

## Active Expiration

To handle keys that are never accessed again, Redis runs an active expiration cycle in its server cron (executed every `hz` times per second, default 10Hz):

### The Active Expiry Algorithm

```text
Every 1/hz seconds:
1. Repeat up to ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP (20) times:
   a. Pick 20 random keys from the expires dict
   b. Delete any that have expired
   c. Track the ratio: expired / sampled
2. If expired ratio > 25%, repeat the cycle immediately
3. Stop if elapsed time exceeds the time budget (1ms per cycle in slow mode)
```

The 25% threshold ensures that when a large number of keys expire simultaneously, Redis keeps cycling until the expiry rate drops below one-quarter of sampled keys.

### Fast vs Slow Expiry Cycles

Redis runs two types of active expiry cycles:

| Mode  | Frequency        | Time Budget | Trigger                    |
|-------|------------------|-------------|----------------------------|
| Fast  | Before each poll | 1ms         | Event loop iteration start |
| Slow  | Every 1/hz secs  | 25ms        | Server cron                |

The fast cycle runs before each `epoll_wait` call to quickly clean up recently-expired keys. The slow cycle runs periodically with a larger budget.

## Configuring the Expiry Cycle

The `hz` configuration controls how frequently the server cron runs:

```bash
redis-cli CONFIG GET hz
```

```text
hz: 10
```

Increase `hz` to expire keys faster at the cost of higher CPU usage:

```bash
redis-cli CONFIG SET hz 20
```

For adaptive behavior, enable `dynamic-hz`:

```bash
redis-cli CONFIG SET dynamic-hz yes
```

With `dynamic-hz`, Redis increases the `hz` value automatically when there are more connected clients, improving responsiveness under load.

## Expiration in Replicas

Replicas do not independently expire keys. Instead:

1. The primary expires the key (lazy or active)
2. The primary sends a `DEL` command to all replicas
3. Replicas apply the `DEL` command to stay in sync

This ensures consistency: a replica never serves a key that the primary has already expired.

However, before the primary sends the `DEL`, a replica may still return a stale value for a key that has logically expired. To prevent this, replicas check the `expires` dict on read and return NULL if the key is expired locally, even before receiving the `DEL`:

```text
Replica receives GET expired_key:
  -> expireIfNeeded("expired_key") - checks local expires dict
  -> key is expired -> return NULL
  -> (primary will send DEL shortly)
```

## Memory Impact of Expired Keys

Even though a key is logically expired after its TTL, it still occupies memory until lazy or active expiry deletes it. In the worst case, Redis might hold many megabytes of expired data.

Monitor expired key statistics:

```bash
redis-cli INFO stats | grep expired
```

```text
expired_keys:1250000
expired_stale_perc:1.23
expired_time_cap_reached_count:42
```

- `expired_keys`: total keys expired since server start
- `expired_stale_perc`: percentage of keys in the expires dict that are already expired
- `expired_time_cap_reached_count`: times the expiry cycle hit its time budget limit

A high `expired_stale_perc` means Redis is falling behind on expiry - consider increasing `hz`.

## Bulk TTL Setting to Spread Expiry Load

If you set TTLs on many keys at once with the same expiry time, they all expire simultaneously, causing a thundering herd on the expiry cycle. Spread expirations using random jitter:

```python
import redis
import random

r = redis.Redis(host='localhost', port=6379)
pipe = r.pipeline()

for i in range(10000):
    ttl = 3600 + random.randint(-300, 300)  # 3600s +/- 5 minutes
    pipe.set(f"session:{i}", f"data_{i}", ex=ttl)

pipe.execute()
```

## Inspecting the Expires Dict

You can get the number of keys with TTLs set:

```bash
redis-cli DEBUG OBJECT mykey
```

```text
Value at:0x7f... refcount:1 encoding:embstr serializedlength:8 lru:12345678 lru_seconds_idle:42 type:string
```

For database-level stats:

```bash
redis-cli INFO keyspace
```

```text
db0:keys=500000,expires=120000,avg_ttl=3480000
```

`expires=120000` means 120,000 keys in database 0 have an expiry set.

## Summary

Redis uses a two-pronged expiration strategy. Lazy expiration handles keys at access time with zero background overhead, while active expiration runs probabilistic cycles to reclaim memory from keys that are never accessed again. The active expiry algorithm samples random keys and repeats aggressively when the expired ratio is high. Understanding this model helps you configure `hz` appropriately, avoid TTL thundering herds, and diagnose high `expired_stale_perc` values before they impact memory usage.
