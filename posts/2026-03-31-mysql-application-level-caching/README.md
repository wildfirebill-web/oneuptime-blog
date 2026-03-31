# How to Implement Application-Level Caching for MySQL Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Cache, Performance

Description: Learn how to implement application-level caching for MySQL queries using in-process memory caches, memoization decorators, and TTL-based eviction strategies.

---

Application-level caching stores query results in the application's own memory or in a local cache layer, reducing round-trips to MySQL for repeated reads. Unlike external caches like Redis, application-level caches have zero network overhead and can use language-native data structures for maximum speed.

## In-Process Dictionary Cache

The simplest approach uses a Python dictionary as a cache with manual TTL management:

```python
import time
import mysql.connector

_cache = {}
db = mysql.connector.connect(host='localhost', database='myapp', user='app', password='secret')

def get_config(key, ttl=300):
    now = time.time()

    # Check cache and TTL
    if key in _cache:
        value, expires_at = _cache[key]
        if now < expires_at:
            return value
        del _cache[key]

    # Cache miss: query MySQL
    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT value FROM config WHERE key_name = %s', (key,))
    row = cursor.fetchone()

    if row:
        _cache[key] = (row['value'], now + ttl)
        return row['value']

    return None
```

This works well for small, infrequently changing configuration data.

## Memoization with functools.lru_cache

For functions with deterministic inputs, Python's `lru_cache` provides built-in memoization:

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def get_category(category_id):
    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT id, name, slug FROM categories WHERE id = %s', (category_id,))
    return cursor.fetchone()
```

`lru_cache` evicts the least recently used entries when the cache reaches `maxsize`. Note that it does not support TTL natively - cached results persist until the process restarts or the entry is evicted by size.

## TTL-Aware Memoization Decorator

Build a decorator that adds TTL support:

```python
import functools

def cache_with_ttl(ttl=60):
    def decorator(func):
        cache = {}

        @functools.wraps(func)
        def wrapper(*args):
            key = args
            now = time.time()

            if key in cache:
                value, expires = cache[key]
                if now < expires:
                    return value

            result = func(*args)
            cache[key] = (result, now + ttl)
            return result

        wrapper.cache_clear = lambda: cache.clear()
        return wrapper
    return decorator


@cache_with_ttl(ttl=120)
def get_product_price(product_id):
    cursor = db.cursor()
    cursor.execute('SELECT price FROM products WHERE id = %s', (product_id,))
    row = cursor.fetchone()
    return float(row[0]) if row else None
```

Call `get_product_price.cache_clear()` after a price update to invalidate the cache.

## Batching Lookups with a Cache

Reduce individual queries by looking up multiple IDs at once and caching each result:

```python
def get_products_bulk(product_ids):
    result = {}
    missing_ids = []

    # Check cache for each ID
    for pid in product_ids:
        cached = _cache.get(f'product:{pid}')
        if cached and time.time() < cached[1]:
            result[pid] = cached[0]
        else:
            missing_ids.append(pid)

    if not missing_ids:
        return result

    # Batch query for missing IDs
    placeholders = ','.join(['%s'] * len(missing_ids))
    cursor = db.cursor(dictionary=True)
    cursor.execute(f'SELECT * FROM products WHERE id IN ({placeholders})', missing_ids)

    for row in cursor.fetchall():
        _cache[f'product:{row["id"]}'] = (row, time.time() + 300)
        result[row['id']] = row

    return result
```

## Cache Size Limits

Use `cachetools` for production-ready LRU and TTL caches:

```python
from cachetools import TTLCache

product_cache = TTLCache(maxsize=1000, ttl=300)

def get_product(product_id):
    if product_id in product_cache:
        return product_cache[product_id]

    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT * FROM products WHERE id = %s', (product_id,))
    product = cursor.fetchone()

    if product:
        product_cache[product_id] = product

    return product
```

## Summary

Application-level caching for MySQL stores query results in process memory using dictionaries, `lru_cache`, or libraries like `cachetools`. This eliminates network overhead compared to Redis but is limited to single-process scope. It is best for read-heavy, low-cardinality data like configuration, categories, and reference tables.
