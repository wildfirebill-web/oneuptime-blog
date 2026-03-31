# How to Implement Stale-While-Revalidate Caching with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Strategy, Performance, Background Job

Description: Implement the stale-while-revalidate caching pattern with Redis to serve fast responses while refreshing data asynchronously in the background.

---

The stale-while-revalidate (SWR) pattern serves cached data immediately - even if it's slightly out of date - while triggering a background refresh. Users get a fast response every time, and the cache is always replenished before the next request.

## How It Works

Each cache entry tracks two expiration times:

1. **Fresh period** - data is current, serve it directly.
2. **Stale period** - data is old but still acceptable. Serve it immediately and refresh in the background.
3. **Expired** - data is too old. Block and fetch fresh data.

```text
|--- fresh (0-60s) ---|--- stale (60-120s) ---|--- expired (120s+) ---|
     serve directly         serve + refresh          block + fetch
```

## Store TTL Metadata Alongside Cached Data

Since Redis only supports a single TTL per key, encode both timestamps in the value:

```python
import redis
import json
import time
import threading

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def swr_set(key: str, value: dict, fresh_ttl: int, stale_ttl: int):
    """Store value with fresh and stale expiry timestamps."""
    entry = {
        "data": value,
        "fresh_until": time.time() + fresh_ttl,
        "stale_until": time.time() + stale_ttl,
    }
    # Store with the full stale TTL so Redis expires it after the stale window
    r.set(key, json.dumps(entry), ex=stale_ttl)

def swr_get(key: str, fetch_fn, fresh_ttl: int = 60, stale_ttl: int = 120):
    """
    Return data immediately. Trigger background refresh if stale.
    Block and fetch if expired.
    """
    raw = r.get(key)

    if raw is None:
        # Cache miss - fetch synchronously
        data = fetch_fn()
        swr_set(key, data, fresh_ttl, stale_ttl)
        return data

    entry = json.loads(raw)
    now = time.time()

    if now < entry["fresh_until"]:
        # Fresh - return immediately
        return entry["data"]

    if now < entry["stale_until"]:
        # Stale - return immediately and refresh in background
        threading.Thread(
            target=_background_refresh,
            args=(key, fetch_fn, fresh_ttl, stale_ttl),
            daemon=True,
        ).start()
        return entry["data"]

    # Expired - fetch synchronously (fallback)
    data = fetch_fn()
    swr_set(key, data, fresh_ttl, stale_ttl)
    return data

def _background_refresh(key: str, fetch_fn, fresh_ttl: int, stale_ttl: int):
    try:
        data = fetch_fn()
        swr_set(key, data, fresh_ttl, stale_ttl)
    except Exception as e:
        print(f"Background refresh failed for {key}: {e}")
```

## Use the SWR Cache in an Application

```python
def fetch_top_products() -> list:
    """Simulate a slow database query."""
    import time as t
    t.sleep(0.1)  # Simulate DB latency
    return [{"id": 1, "name": "Widget"}, {"id": 2, "name": "Gadget"}]

def get_top_products() -> list:
    return swr_get(
        key="products:top",
        fetch_fn=fetch_top_products,
        fresh_ttl=60,    # Fresh for 60 seconds
        stale_ttl=120,   # Serve stale up to 120 seconds
    )
```

## Prevent Thundering Herd During Refresh

Use a Redis lock to ensure only one background worker refreshes a key at a time:

```python
def _background_refresh(key: str, fetch_fn, fresh_ttl: int, stale_ttl: int):
    lock_key = f"{key}:refresh_lock"
    acquired = r.set(lock_key, "1", nx=True, ex=10)

    if not acquired:
        return  # Another worker is already refreshing

    try:
        data = fetch_fn()
        swr_set(key, data, fresh_ttl, stale_ttl)
    finally:
        r.delete(lock_key)
```

## Verify the Pattern Works

```bash
# First request - cache miss, synchronous fetch
# Subsequent requests within 60s - fresh, instant response
# Requests between 60-120s - stale, instant response + background refresh
# Requests after 120s - synchronous fetch again
redis-cli ttl "products:top"
# Returns remaining seconds in stale window
```

## Summary

Stale-while-revalidate balances freshness and performance by serving cached data immediately and refreshing it in the background. The key implementation detail is storing both the fresh and stale expiry timestamps alongside the data so you can make the right decision at read time. Use a Redis lock during background refresh to prevent multiple simultaneous fetches for the same key.
