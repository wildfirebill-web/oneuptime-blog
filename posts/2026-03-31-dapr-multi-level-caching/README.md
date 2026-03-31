# How to Implement Multi-Level Caching with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cache, Multi-Level, State Management, Performance

Description: Learn how to implement a multi-level (L1/L2) caching strategy with Dapr using an in-process local cache and a shared Dapr state store for distributed caching.

---

## What is Multi-Level Caching?

Multi-level caching uses two cache tiers:
- **L1 (local cache):** An in-process cache within each service instance. Extremely fast but not shared. Used for the most frequently accessed, rarely changed data.
- **L2 (distributed cache):** A shared cache backed by Redis via Dapr state management. Shared across all instances. Slower than L1 but much faster than the database.

Reads check L1 first, fall through to L2 on a miss, and finally query the database if both miss.

## Implementing L1 Cache with an LRU Cache

Use an in-process LRU cache for L1. In Python, `cachetools` provides a simple TTL-aware LRU:

```python
from cachetools import TTLCache
import threading

# L1: in-process cache, 500 entries, 30-second TTL
l1_cache = TTLCache(maxsize=500, ttl=30)
l1_lock = threading.Lock()

def get_from_l1(key: str):
    with l1_lock:
        return l1_cache.get(key)

def set_in_l1(key: str, value):
    with l1_lock:
        l1_cache[key] = value

def invalidate_l1(key: str):
    with l1_lock:
        l1_cache.pop(key, None)
```

## Implementing L2 Cache with Dapr State Management

```python
import httpx
import json

DAPR_HTTP_PORT = 3500
L2_STORE = "redis-cache"

async def get_from_l2(key: str):
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/{L2_STORE}/{key}"
        )
        if resp.status_code == 200 and resp.text:
            return resp.json()
    return None

async def set_in_l2(key: str, value: dict, ttl_seconds: int = 300):
    async with httpx.AsyncClient() as client:
        await client.post(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/{L2_STORE}",
            json=[{
                "key": key,
                "value": value,
                "metadata": {"ttlInSeconds": str(ttl_seconds)}
            }]
        )
```

## Unified Multi-Level Cache Read

```python
async def get_product(product_id: str) -> dict:
    cache_key = f"product:{product_id}"

    # L1 check (in-process, no network)
    l1_result = get_from_l1(cache_key)
    if l1_result is not None:
        return {**l1_result, "cache_level": "L1"}

    # L2 check (Dapr/Redis, single network hop)
    l2_result = await get_from_l2(cache_key)
    if l2_result is not None:
        # Backfill L1 from L2
        set_in_l1(cache_key, l2_result)
        return {**l2_result, "cache_level": "L2"}

    # Database read (slowest, only on full miss)
    product = await db.fetch_product(product_id)
    if product:
        # Populate both cache levels
        set_in_l1(cache_key, product)
        await set_in_l2(cache_key, product, ttl_seconds=300)

    return {**product, "cache_level": "DB"} if product else None
```

## Cache Invalidation Across Levels

When a product is updated, invalidate both L1 and L2. For L1 across multiple instances, use Dapr pub/sub to broadcast the invalidation:

```python
async def update_product(product_id: str, product: dict):
    await db.save_product(product_id, product)

    # Invalidate local L1
    invalidate_l1(f"product:{product_id}")

    # Invalidate L2
    async with httpx.AsyncClient() as client:
        await client.delete(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/{L2_STORE}/product:{product_id}"
        )
        # Broadcast L1 invalidation to all instances
        await client.post(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0/publish/pubsub/cache-invalidation",
            json={"key": f"product:{product_id}"}
        )
```

All service instances subscribe to `cache-invalidation` and call `invalidate_l1` when they receive the event.

## Summary

Multi-level caching with Dapr combines the speed of in-process L1 caching with the consistency of a Dapr-managed distributed L2 cache. L1 serves repeated requests within the same instance with zero network overhead, while L2 prevents database hits from occurring in multiple instances simultaneously. Dapr pub/sub ties the two levels together by broadcasting L1 invalidation events across all running instances.
