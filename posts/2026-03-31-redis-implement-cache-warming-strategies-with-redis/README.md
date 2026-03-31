# How to Implement Cache Warming Strategies with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Cache Warming, Performance, Strategy

Description: Learn how to warm up your Redis cache before traffic hits by pre-loading hot data from the database using batch loading, lazy warming, and scheduled strategies.

---

## What is Cache Warming?

Cache warming is the process of pre-populating a Redis cache with data before users start requesting it. A cold cache results in all requests going to the database until entries are populated, causing latency spikes at startup, after a Redis restart, or after a deployment.

Cache warming strategies balance startup time against the risk of serving stale or missing data during high traffic.

## Why Cache Warming Matters

```text
Without warming:
  Startup -> 100% cache misses -> DB overload -> slow responses

With warming:
  Pre-load popular data -> Startup -> high cache hit rate -> fast responses
```

## Strategy 1 - Eager (Full) Warm at Startup

Load all frequently accessed data into Redis before the application starts serving traffic.

```python
import redis
import json
from typing import List

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def warm_all_products() -> int:
    """Load all active products into Redis cache at startup."""
    products = db_fetch_all_active_products()
    pipe = r.pipeline()

    for product in products:
        key = f"product:{product['id']}"
        pipe.setex(key, 3600, json.dumps(product))

    pipe.execute()
    return len(products)

def warm_all_users() -> int:
    """Load the most recently active users."""
    users = db_fetch_recent_users(limit=10_000)
    pipe = r.pipeline()

    for user in users:
        key = f"user:{user['id']}"
        pipe.setex(key, 1800, json.dumps(user))

    pipe.execute()
    return len(users)

# At application startup
if __name__ == '__main__':
    product_count = warm_all_products()
    user_count    = warm_all_users()
    print(f"Warmed {product_count} products, {user_count} users")
    start_server()
```

## Strategy 2 - Lazy Warming (Cache-Aside)

Let the cache warm itself organically as users make requests. Each miss loads data and stores it for subsequent requests.

```python
def get_product(product_id: int) -> dict:
    """Lazy warming - populate cache on first access."""
    key = f"product:{product_id}"

    cached = r.get(key)
    if cached:
        return json.loads(cached)

    # Cache miss - load from DB and warm the cache
    product = db_fetch_product(product_id)
    if product:
        r.setex(key, 3600, json.dumps(product))

    return product
```

This requires no upfront work but leaves the cache cold after restarts until organic traffic repopulates it.

## Strategy 3 - Warm from Access Logs (Hot Keys)

Analyze access logs to identify the most popular keys and pre-warm only those:

```python
import json
from collections import Counter

def extract_hot_keys(log_file: str, top_n: int = 1000) -> List[str]:
    """Extract the most requested keys from access logs."""
    counter = Counter()
    with open(log_file) as f:
        for line in f:
            entry = json.loads(line)
            counter[entry['key']] += 1

    return [key for key, _ in counter.most_common(top_n)]

def warm_from_hot_keys(log_file: str) -> None:
    hot_keys = extract_hot_keys(log_file, top_n=500)
    pipe = r.pipeline()

    for key in hot_keys:
        # key format: "product:123"
        entity_type, entity_id = key.split(':')

        if entity_type == 'product':
            data = db_fetch_product(int(entity_id))
            if data:
                pipe.setex(key, 3600, json.dumps(data))

    pipe.execute()
    print(f"Pre-warmed {len(hot_keys)} hot keys from access logs")
```

## Strategy 4 - Scheduled Warm via Cron

Use a background job to periodically refresh expiring hot data:

```python
import schedule
import time

def warm_scheduled() -> None:
    """Refresh keys that are expiring within the next 5 minutes."""
    hot_keys = get_hot_keys_list()  # maintained separately
    pipe = r.pipeline()
    for key in hot_keys:
        pipe.ttl(key)
    ttls = pipe.execute()

    # Find keys with less than 300 seconds remaining
    expiring = [k for k, t in zip(hot_keys, ttls) if 0 < t < 300]

    for key in expiring:
        entity_type, entity_id = key.split(':')
        if entity_type == 'product':
            data = db_fetch_product(int(entity_id))
            if data:
                r.setex(key, 3600, json.dumps(data))

    print(f"Refreshed {len(expiring)} expiring keys")

# Schedule to run every 2 minutes
schedule.every(2).minutes.do(warm_scheduled)

def run_scheduler():
    while True:
        schedule.run_pending()
        time.sleep(1)
```

## Strategy 5 - Warm from a Snapshot

Dump cache contents to a file and restore on next startup:

```python
def export_cache_snapshot(pattern: str = 'product:*', output_file: str = '/tmp/cache_snapshot.json') -> None:
    """Export current cache to a snapshot file."""
    snapshot = {}
    cursor = 0

    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=500)
        if keys:
            pipe = r.pipeline()
            for key in keys:
                pipe.get(key)
            values = pipe.execute()
            for key, value in zip(keys, values):
                if value:
                    snapshot[key] = value
        if cursor == 0:
            break

    with open(output_file, 'w') as f:
        json.dump(snapshot, f)

    print(f"Exported {len(snapshot)} keys to snapshot")

def restore_from_snapshot(snapshot_file: str = '/tmp/cache_snapshot.json', ttl: int = 3600) -> None:
    """Restore cache from snapshot on startup."""
    with open(snapshot_file) as f:
        snapshot = json.load(f)

    pipe = r.pipeline()
    for key, value in snapshot.items():
        pipe.setex(key, ttl, value)
    pipe.execute()

    print(f"Restored {len(snapshot)} keys from snapshot")
```

## Combining Strategies

A practical approach for production:

```python
def warm_on_startup():
    # 1. Restore from snapshot (fastest - < 1 second)
    try:
        restore_from_snapshot('/tmp/cache_snapshot.json')
    except FileNotFoundError:
        pass

    # 2. Eager-warm the most critical data (product catalog)
    warm_all_products()

    # 3. Let everything else warm lazily via cache-aside reads
    print("Cache warming complete - server ready")
```

## Summary

Cache warming prevents cold start latency spikes by pre-loading Redis with data before traffic arrives. Choose your strategy based on dataset size and freshness requirements: eager warming loads everything at startup for maximum hit rate, lazy warming is simple but leaves the cache cold after restarts, hot-key warming targets the most popular data for efficiency, and scheduled warming keeps expiring hot keys fresh. Most production systems combine snapshot restoration for speed with selective eager warming for critical data.
