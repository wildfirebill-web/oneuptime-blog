# How to Implement Refresh-Ahead Cache Pattern with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Refresh-Ahead, Cache Patterns, Ttl

Description: Learn how to implement the refresh-ahead cache pattern with Redis to proactively refresh cache entries before they expire, eliminating cache miss latency.

---

## What is the Refresh-Ahead Pattern?

In the refresh-ahead pattern, the cache layer proactively refreshes entries before their TTL expires, rather than waiting for a request to discover a stale or missing key. When a read detects that a cache entry is near its expiration, a background refresh is triggered immediately while the current (still-valid) cached value is returned.

```text
Request arrives
     |
     v
  Cache entry found
     |
  Near expiry? --> Yes --> Trigger async refresh
     |                         |
  Return cached value     Load from DB in background
                               |
                          Update cache
```

## When to Use Refresh-Ahead

- High-read workloads where cache misses are expensive (slow DB queries)
- Data with predictable access patterns (popular product pages, dashboard metrics)
- When you need consistent low latency with no "thundering herd" on TTL expiry
- Configuration values, feature flags, reference data

## Core Concept: Two TTLs

The refresh-ahead pattern uses two time values:
- **Cache TTL** - how long the data is stored in Redis
- **Refresh threshold** - what fraction of TTL remaining triggers a refresh

```text
|------ Cache TTL (e.g. 60s) ------|
|-----------------|
  Refresh zone    ^ At this point, background refresh fires
(e.g. at 30% TTL remaining)
```

## Python Implementation

```python
import redis
import json
import threading
import time
from typing import Callable, Optional, Any

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

CACHE_TTL_SECONDS  = 60
REFRESH_THRESHOLD  = 0.3  # Refresh when 30% of TTL remains

def get_refresh_ahead(
    key: str,
    loader: Callable[[], Any],
    ttl: int = CACHE_TTL_SECONDS,
    threshold: float = REFRESH_THRESHOLD
) -> Optional[Any]:
    """Read-through with refresh-ahead: refresh before expiry."""
    cached   = r.get(key)
    ttl_left = r.ttl(key)

    if cached is not None:
        # Check if we are in the refresh zone
        if ttl_left < ttl * threshold:
            _trigger_async_refresh(key, loader, ttl)
        return json.loads(cached)

    # Cache miss - load synchronously
    value = loader()
    if value is not None:
        r.setex(key, ttl, json.dumps(value))
    return value

def _trigger_async_refresh(key: str, loader: Callable, ttl: int) -> None:
    """Trigger a background refresh only if not already in progress."""
    lock_key = f"refresh_lock:{key}"
    acquired = r.set(lock_key, 1, nx=True, ex=ttl)  # Lock for one TTL period

    if acquired:
        thread = threading.Thread(
            target=_do_refresh,
            args=(key, loader, ttl, lock_key),
            daemon=True
        )
        thread.start()

def _do_refresh(key: str, loader: Callable, ttl: int, lock_key: str) -> None:
    try:
        value = loader()
        if value is not None:
            r.setex(key, ttl, json.dumps(value))
    except Exception as e:
        print(f"Background refresh failed for {key}: {e}")
    finally:
        r.delete(lock_key)
```

### Usage

```python
def load_product_from_db(product_id: int) -> dict:
    # Simulate a slow DB query
    return {"id": product_id, "name": "Widget", "price": 9.99, "stock": 100}

def get_product(product_id: int) -> dict:
    key = f"product:{product_id}"
    return get_refresh_ahead(
        key,
        loader=lambda: load_product_from_db(product_id),
        ttl=60,
        threshold=0.3
    )

# First call - cache miss, loads from DB
p = get_product(42)
print(p)  # {"id": 42, "name": "Widget", ...}

# Subsequent calls hit the cache; when TTL drops below 18s (30% of 60),
# a background refresh fires and the caller still gets the cached value instantly
```

## Node.js Implementation

```javascript
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });

const CACHE_TTL = 60;
const REFRESH_THRESHOLD = 0.3;

async function getRefreshAhead(key, loader, ttl = CACHE_TTL, threshold = REFRESH_THRESHOLD) {
  const [cached, ttlLeft] = await Promise.all([
    client.get(key),
    client.ttl(key)
  ]);

  if (cached !== null) {
    if (ttlLeft < ttl * threshold) {
      triggerAsyncRefresh(key, loader, ttl);
    }
    return JSON.parse(cached);
  }

  // Cache miss - synchronous load
  const value = await loader();
  if (value != null) {
    await client.setEx(key, ttl, JSON.stringify(value));
  }
  return value;
}

async function triggerAsyncRefresh(key, loader, ttl) {
  const lockKey = `refresh_lock:${key}`;
  const acquired = await client.set(lockKey, '1', { NX: true, EX: ttl });
  if (!acquired) return;

  try {
    const value = await loader();
    if (value != null) {
      await client.setEx(key, ttl, JSON.stringify(value));
    }
  } catch (err) {
    console.error(`Background refresh failed for ${key}:`, err);
  } finally {
    await client.del(lockKey);
  }
}

// Usage
async function getPopularItem(itemId) {
  return getRefreshAhead(
    `item:${itemId}`,
    () => db.findById('items', itemId),
    120, // 2-minute TTL
    0.25 // refresh at 25% remaining
  );
}
```

## Monitoring Refresh Activity

Log when refreshes occur to tune threshold values:

```python
import logging

log = logging.getLogger(__name__)

def _do_refresh(key: str, loader: Callable, ttl: int, lock_key: str) -> None:
    start = time.monotonic()
    try:
        value = loader()
        elapsed = time.monotonic() - start
        if value is not None:
            r.setex(key, ttl, json.dumps(value))
            log.info("Refresh-ahead completed", extra={
                "key": key,
                "duration_ms": round(elapsed * 1000, 2)
            })
    except Exception as e:
        log.error("Refresh-ahead failed for %s: %s", key, e)
    finally:
        r.delete(lock_key)
```

## Summary

The refresh-ahead pattern eliminates cold cache latency by proactively refreshing entries before they expire. When a read detects that a cache key is within the refresh threshold (e.g., 30% TTL remaining), it triggers a background reload while immediately returning the still-valid cached value. A distributed lock prevents multiple nodes from refreshing the same key simultaneously, and callers never experience a cache miss once the key is initially populated.
