# How to Implement Cache Preloading at Application Startup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Performance

Description: Implement Redis cache preloading at application startup to eliminate cold-start latency and serve warm cache from the first request.

---

A cold cache causes the first wave of requests after a deployment to hit your database directly, causing latency spikes. Cache preloading (also called cache warming) loads frequently accessed data into Redis during startup so your application is ready to serve fast responses from the very first request.

## When to Use Cache Preloading

- High-traffic endpoints with predictable data sets (product catalogs, config values, feature flags)
- Applications that deploy frequently and cannot tolerate cold-start slowdowns
- Read-heavy workloads where a temporary cache miss storm would overwhelm the database

## Basic Preloading Pattern

```python
import redis
import json
import logging
from typing import List

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
logger = logging.getLogger(__name__)

def preload_products():
    logger.info("Preloading product catalog into Redis...")
    products = fetch_all_active_products_from_db()  # DB call at startup
    pipe = r.pipeline()
    for product in products:
        key = f"product:{product['id']}"
        pipe.setex(key, 3600, json.dumps(product))
    pipe.execute()
    logger.info(f"Preloaded {len(products)} products into Redis cache.")

def startup():
    preload_products()
    preload_feature_flags()
    preload_config_values()
```

## Preloading with Batching

For large datasets, use pipelining to load data in chunks instead of individual SET calls:

```python
def preload_in_batches(items: List[dict], key_prefix: str, ttl: int, batch_size: int = 500):
    pipe = r.pipeline()
    count = 0
    for item in items:
        key = f"{key_prefix}:{item['id']}"
        pipe.setex(key, ttl, json.dumps(item))
        count += 1
        if count % batch_size == 0:
            pipe.execute()
            pipe = r.pipeline()
    if count % batch_size != 0:
        pipe.execute()
    return count
```

## Async Preloading (Non-Blocking)

For applications that must start quickly, preload in a background thread:

```python
import threading

def warm_cache_async():
    thread = threading.Thread(target=preload_all, daemon=True)
    thread.start()
    logger.info("Cache warming started in background thread.")

def preload_all():
    try:
        preload_products()
        preload_config_values()
        logger.info("Cache warming complete.")
    except Exception as e:
        logger.error(f"Cache warming failed: {e}")
```

## Health Check Gate

Block traffic until the cache is warm using a readiness flag:

```python
CACHE_READY_KEY = "app:cache_ready"

def mark_cache_ready():
    r.set(CACHE_READY_KEY, "1", ex=86400)

def is_cache_ready() -> bool:
    return r.exists(CACHE_READY_KEY) == 1

# In your health/readiness endpoint
def readiness_check():
    if not is_cache_ready():
        return {"status": "warming"}, 503
    return {"status": "ok"}, 200
```

## Verifying Preloaded Data

```bash
# Check number of keys loaded
redis-cli DBSIZE

# Inspect a specific preloaded key
redis-cli GET "product:42"

# Check TTL on a preloaded key
redis-cli TTL "product:42"
```

## Summary

Cache preloading at startup turns cold-start latency from a production risk into a non-issue. By loading critical data into Redis before traffic arrives, your application serves warm cached results from the first request. Using pipelining for bulk loads and a readiness flag to gate traffic keeps the process efficient and safe under real deployment conditions.
