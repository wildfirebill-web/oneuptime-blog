# Why You Should Not Use Short TTLs on Frequently Accessed Keys

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, TTL, Cache

Description: Learn why overly aggressive TTLs on hot keys cause cache stampedes, database overload, and high miss rates - and how to tune TTLs correctly.

---

Setting short TTLs feels safe - it ensures data freshness. But when those keys are hot and expire simultaneously, you create a cache stampede: hundreds of requests hit your database at once. This is one of the most common causes of database overload in systems that use Redis caching.

## The Cache Stampede Problem

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Anti-pattern: very short TTL on a hot key
def get_product_price_naive(product_id: str) -> float:
    cached = r.get(f"price:{product_id}")
    if cached:
        return float(cached)

    # Cache miss: fetch from database
    price = fetch_price_from_db(product_id)
    r.setex(f"price:{product_id}", 5, price)  # 5 second TTL - too short!
    return price

def fetch_price_from_db(product_id: str) -> float:
    return 9.99  # Simulated DB call
```

With 500 concurrent requests and a 5-second TTL, you get a DB thundering herd every 5 seconds.

## Solution 1: Longer TTLs with Background Refresh

Use a longer TTL and refresh the cache proactively before it expires:

```python
CACHE_TTL = 300        # 5 minutes
REFRESH_THRESHOLD = 60  # Refresh when < 60 seconds remain

def get_product_price(product_id: str) -> float:
    key = f"price:{product_id}"
    pipe = r.pipeline(transaction=False)
    pipe.get(key)
    pipe.ttl(key)
    value, ttl = pipe.execute()

    if value is not None:
        # Proactively refresh in background when TTL is low
        if ttl < REFRESH_THRESHOLD:
            import threading
            threading.Thread(target=refresh_cache, args=(product_id,), daemon=True).start()
        return float(value)

    # Cache miss
    price = fetch_price_from_db(product_id)
    r.setex(key, CACHE_TTL, price)
    return price

def refresh_cache(product_id: str):
    price = fetch_price_from_db(product_id)
    r.setex(f"price:{product_id}", CACHE_TTL, price)
```

## Solution 2: Probabilistic Early Expiration (XFetch)

Add random jitter near expiry so different servers refresh at different times:

```python
import math
import random

def get_with_xfetch(key: str, beta: float = 1.0):
    """XFetch algorithm: probabilistic early expiration."""
    data = r.get(key)
    ttl = r.ttl(key)

    if data is None:
        return None, 0, 0

    # Probability of early refresh increases as TTL decreases
    should_refresh = (-beta * math.log(random.random())) > ttl
    return data, ttl, should_refresh
```

## Solution 3: TTL Jitter

When many keys are set simultaneously, add random jitter to spread out expiration:

```python
import random

def set_with_jitter(key: str, value: str, base_ttl: int, jitter_pct: float = 0.1):
    """Spread expiration by +/- jitter_pct of base_ttl."""
    jitter = int(base_ttl * jitter_pct * random.uniform(-1, 1))
    actual_ttl = base_ttl + jitter
    r.setex(key, max(1, actual_ttl), value)
```

## Solution 4: Mutex on Cache Miss

Use a lock to ensure only one process fetches from the DB on a miss:

```python
def get_with_mutex(key: str, fetcher, ttl: int = 300) -> str | None:
    value = r.get(key)
    if value:
        return value

    lock_key = f"lock:{key}"
    if r.set(lock_key, 1, ex=5, nx=True):
        try:
            value = fetcher()
            r.setex(key, ttl, value)
        finally:
            r.delete(lock_key)
    else:
        # Another process is fetching - wait briefly
        time.sleep(0.05)
        return r.get(key)  # May still be None if fetcher is slow

    return value
```

## Choosing the Right TTL

```text
Static reference data (countries, currencies): 1-24 hours
Product catalog: 5-30 minutes with background refresh
User profile: 15-60 minutes with cache invalidation on write
Computed aggregates (totals, counts): 1-5 minutes with jitter
Real-time prices / inventory: use cache invalidation, not TTL
```

## Summary

Short TTLs on hot keys are a leading cause of cache stampedes and database overload. Use longer TTLs with proactive background refresh, add TTL jitter to spread expiration across your fleet, and apply mutex-based miss protection for especially expensive queries. These patterns keep your database healthy even at peak traffic.
