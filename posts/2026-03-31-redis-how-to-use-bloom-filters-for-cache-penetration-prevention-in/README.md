# How to Use Bloom Filters for Cache Penetration Prevention in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, Cache Penetration, Security, Caching

Description: Learn how to use Redis Bloom filters to prevent cache penetration attacks where requests for non-existent keys bypass the cache and hammer your database.

---

## What Is Cache Penetration

Cache penetration occurs when requests are made for data that does not exist in either the cache or the database. Each request misses the cache and queries the database, returning null. Attackers can exploit this by flooding an API with random IDs, bypassing the cache entirely and causing database overload.

## How Bloom Filters Prevent It

A Bloom filter stores all valid keys that exist in the database. Before querying the cache or database, check if the key is in the Bloom filter. If the filter says "no" (definite negative), return 404 immediately without touching the database. If the filter says "yes" (probable positive), proceed normally.

## Setup

```bash
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Initializing the Bloom Filter from Existing Data

```python
from redis import Redis
import psycopg2  # or any database driver

r = Redis(decode_responses=True)

def init_product_bloom(db_conn, error_rate: float = 0.001,
                       capacity: int = 10_000_000):
    key = "bloom:valid_products"
    try:
        r.bf().info(key)
        return  # Already initialized
    except Exception:
        r.bf().reserve(key, error_rate, capacity)

    # Load all valid product IDs from database
    cursor = db_conn.cursor()
    cursor.execute("SELECT id FROM products WHERE active = true")
    batch = []
    for (product_id,) in cursor:
        batch.append(str(product_id))
        if len(batch) >= 10000:
            r.bf().madd(key, *batch)
            batch.clear()
    if batch:
        r.bf().madd(key, *batch)

def might_exist(key: str, item_id: str) -> bool:
    return r.bf().exists(key, item_id)
```

## Cache Lookup with Bloom Filter Guard

```python
import json

def get_product(product_id: int, db_conn) -> dict | None:
    bloom_key = "bloom:valid_products"
    cache_key = f"product:{product_id}"
    str_id = str(product_id)

    # Step 1: Bloom filter check
    if not might_exist(bloom_key, str_id):
        # Definitely doesn't exist - return early, no DB query
        return None

    # Step 2: Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Step 3: Query database (cache miss but Bloom says it might exist)
    cursor = db_conn.cursor()
    cursor.execute("SELECT id, name, price FROM products WHERE id = %s", (product_id,))
    row = cursor.fetchone()

    if row is None:
        # False positive from Bloom filter - cache null result briefly
        r.set(cache_key, "null", ex=60)
        return None

    product = {"id": row[0], "name": row[1], "price": float(row[2])}
    r.set(cache_key, json.dumps(product), ex=3600)
    return product
```

## Keeping the Bloom Filter in Sync

When new products are added or deleted, update the filter:

```python
def on_product_created(product_id: int):
    """Call when a new product is added to the database."""
    r.bf().add("bloom:valid_products", str(product_id))

def on_product_deleted(product_id: int):
    """
    Bloom filters don't support deletion.
    Instead, use a Cuckoo filter if deletions are frequent.
    For rare deletions, a full filter rebuild overnight is acceptable.
    """
    # Option 1: Use Cuckoo filter for deletable sets
    r.cf().delete("cf:valid_products", str(product_id))
    # Option 2: Schedule periodic rebuild from database
    pass

def rebuild_bloom_filter(db_conn):
    """Rebuild the filter nightly from source of truth."""
    new_key = "bloom:valid_products:new"
    r.bf().reserve(new_key, 0.001, 10_000_000)
    cursor = db_conn.cursor()
    cursor.execute("SELECT id FROM products WHERE active = true")
    batch = []
    for (pid,) in cursor:
        batch.append(str(pid))
        if len(batch) >= 10000:
            r.bf().madd(new_key, *batch)
            batch.clear()
    if batch:
        r.bf().madd(new_key, *batch)
    r.rename(new_key, "bloom:valid_products")
```

## FastAPI Integration

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/products/{product_id}")
def get_product_endpoint(product_id: int):
    # Bloom filter check before any DB/cache access
    if not r.bf().exists("bloom:valid_products", str(product_id)):
        raise HTTPException(status_code=404, detail="Product not found")

    cached = r.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)

    # Fetch from DB...
    return {"id": product_id}
```

## Memory Estimate

```text
1 million items,  0.1% false positive rate:  ~1.8 MB
10 million items, 0.1% false positive rate:  ~18 MB
100 million items, 0.1% false positive rate: ~180 MB
```

## Summary

Redis Bloom filters prevent cache penetration by maintaining a memory-efficient set of all valid keys. Requests for non-existent keys are rejected before touching the cache or database. The small rate of false positives (configurable) means some invalid requests still reach the database, but random attack traffic is almost entirely blocked. Periodic filter rebuilds from the database source of truth keep the filter accurate.
