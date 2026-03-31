# How to Handle Cache Stampede with Dapr Distributed Lock

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cache, Distributed Lock, Stampede, Concurrency

Description: Learn how to prevent cache stampede (thundering herd) using Dapr distributed lock to ensure only one instance repopulates the cache after a miss.

---

## What is Cache Stampede?

A cache stampede occurs when a popular cache entry expires and many concurrent requests all experience a miss simultaneously. Each request then queries the database, flooding it with redundant identical queries. Dapr's distributed lock building block solves this by allowing only one request to populate the cache while others wait and reuse the result.

## The Problem Without Locking

Without any protection, an expired popular product entry causes this scenario:

```python
# 100 concurrent requests all see cache miss simultaneously
# All 100 execute the database query at the same time
# Database receives 100 identical queries for the same product
async def get_product(product_id: str) -> dict:
    cached = await get_from_cache(f"product:{product_id}")
    if cached:
        return cached
    # 100 instances reach this point simultaneously
    product = await db.fetch_product(product_id)  # 100x database load
    await set_in_cache(f"product:{product_id}", product, ttl=300)
    return product
```

## Preventing Stampede with Dapr Distributed Lock

The Dapr distributed lock API (`/v1.0-alpha1/lock`) uses Redis (or another lock store) to serialize cache population:

```python
import asyncio
import uuid
import httpx
from typing import Optional

DAPR_HTTP_PORT = 3500
LOCK_STORE = "redislock"
CACHE_STORE = "statestore"

async def acquire_lock(resource_id: str, owner: str, ttl_seconds: int = 10) -> bool:
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0-alpha1/lock/{LOCK_STORE}",
            json={
                "resourceId": resource_id,
                "lockOwner": owner,
                "expiryInSeconds": ttl_seconds
            }
        )
        return resp.json().get("success", False)

async def release_lock(resource_id: str, owner: str):
    async with httpx.AsyncClient() as client:
        await client.post(
            f"http://localhost:{DAPR_HTTP_PORT}/v1.0-alpha1/unlock/{LOCK_STORE}",
            json={"resourceId": resource_id, "lockOwner": owner}
        )

async def get_product_safe(product_id: str) -> Optional[dict]:
    cache_key = f"product:{product_id}"
    lock_key = f"repopulate:product:{product_id}"
    owner = str(uuid.uuid4())

    # Check cache first
    cached = await get_from_cache(cache_key)
    if cached:
        return cached

    # Try to acquire lock for cache population
    acquired = await acquire_lock(lock_key, owner, ttl_seconds=10)

    if acquired:
        try:
            # Check cache again in case another instance populated it
            # while we were acquiring the lock
            cached = await get_from_cache(cache_key)
            if cached:
                return cached

            # We hold the lock - populate the cache
            product = await db.fetch_product(product_id)
            await set_in_cache(cache_key, product, ttl=300)
            return product
        finally:
            await release_lock(lock_key, owner)
    else:
        # Another instance is populating - wait briefly and read from cache
        for attempt in range(5):
            await asyncio.sleep(0.1 * (attempt + 1))
            cached = await get_from_cache(cache_key)
            if cached:
                return cached
        # Fallback to direct database read if cache is still empty
        return await db.fetch_product(product_id)
```

## Configuring the Lock Store Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: redislock
spec:
  type: lock.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    value: ""
```

## Testing the Stampede Prevention

Simulate concurrent requests in a test to verify only one database query runs:

```python
import asyncio

async def test_stampede_prevention():
    query_count = 0
    original_fetch = db.fetch_product

    async def counting_fetch(product_id):
        nonlocal query_count
        query_count += 1
        await asyncio.sleep(0.05)  # Simulate slow query
        return await original_fetch(product_id)

    db.fetch_product = counting_fetch

    # Run 50 concurrent requests
    results = await asyncio.gather(*[get_product_safe("prod-1") for _ in range(50)])

    print(f"Database queries: {query_count}")  # Should be 1, not 50
    assert query_count == 1
    assert all(r is not None for r in results)
```

## Summary

Dapr's distributed lock building block prevents cache stampede by serializing cache population across all service instances. The double-check locking pattern (check cache, acquire lock, check cache again, populate) ensures only one database query runs per cache miss regardless of how many instances are running simultaneously. Combined with Dapr state management for the cache itself, the entire solution uses only Dapr building blocks.
