# How to Implement SKU Availability Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, E-Commerce, Hash, Cache

Description: Cache SKU stock levels and availability status with Redis Hashes - serve availability checks in under 1ms, handle multi-variant products, and sync with inventory on updates.

---

SKU availability checks happen on product pages, search results, and during checkout. Caching availability in Redis eliminates repeated database queries and keeps page loads fast.

## Data Model

```text
sku:{skuId}:availability     -> Hash: stock_count, status, last_updated
product:{productId}:skus     -> Set of SKU IDs for the product
sku:batch:{batchId}          -> Temporary key for batch availability jobs
```

## Caching SKU Availability

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

SKU_CACHE_TTL = 300  # 5 minutes

def cache_sku_availability(sku_id, stock_count, force_status=None):
    if force_status:
        status = force_status
    elif stock_count <= 0:
        status = "out_of_stock"
    elif stock_count <= 5:
        status = "low_stock"
    else:
        status = "in_stock"

    pipe = r.pipeline()
    pipe.hset(f"sku:{sku_id}:availability", mapping={
        "stock_count": str(stock_count),
        "status": status,
        "last_updated": str(time.time()),
    })
    pipe.expire(f"sku:{sku_id}:availability", SKU_CACHE_TTL)
    pipe.execute()

def get_sku_availability(sku_id):
    data = r.hgetall(f"sku:{sku_id}:availability")
    if data:
        data["stock_count"] = int(data.get("stock_count", 0))
    return data
```

## Cache-Aside with Database Fallback

```python
def get_sku_availability_with_fallback(sku_id, db_fetch_fn):
    cached = get_sku_availability(sku_id)
    if cached:
        return cached
    # Fetch from database
    db_data = db_fetch_fn(sku_id)
    if db_data:
        cache_sku_availability(sku_id, db_data["stock_count"])
        return get_sku_availability(sku_id)
    return None
```

## Batch Availability Check

Check availability for multiple SKUs in one round trip:

```python
def batch_sku_availability(sku_ids):
    pipe = r.pipeline()
    for sku_id in sku_ids:
        pipe.hgetall(f"sku:{sku_id}:availability")
    results = pipe.execute()
    availability = {}
    for sku_id, data in zip(sku_ids, results):
        if data:
            data["stock_count"] = int(data.get("stock_count", 0))
            availability[sku_id] = data
        else:
            availability[sku_id] = None  # Cache miss
    return availability
```

## Decrementing Stock on Purchase

```python
def decrement_sku_stock(sku_id, quantity=1):
    key = f"sku:{sku_id}:availability"
    script = """
    local stock = tonumber(redis.call('HGET', KEYS[1], 'stock_count'))
    if stock == nil or stock < tonumber(ARGV[1]) then return 0 end
    local new_stock = stock - tonumber(ARGV[1])
    redis.call('HSET', KEYS[1], 'stock_count', tostring(new_stock))
    if new_stock <= 0 then
        redis.call('HSET', KEYS[1], 'status', 'out_of_stock')
    elseif new_stock <= 5 then
        redis.call('HSET', KEYS[1], 'status', 'low_stock')
    end
    return new_stock
    """
    return r.eval(script, 1, key, quantity)
```

## Multi-Variant Product Availability

```python
def get_product_availability_summary(product_id):
    sku_ids = list(r.smembers(f"product:{product_id}:skus"))
    availability = batch_sku_availability(sku_ids)
    return {
        "skus": availability,
        "any_in_stock": any(
            v and v.get("status") != "out_of_stock"
            for v in availability.values()
        ),
    }
```

## Example Usage

```bash
# Cache SKU availability
HSET sku:SKU-001:availability stock_count 50 status in_stock last_updated 1743360000
EXPIRE sku:SKU-001:availability 300

# Check availability
HGETALL sku:SKU-001:availability

# After purchase
HINCRBY sku:SKU-001:availability stock_count -1
```

## Summary

Redis Hashes cache SKU availability data with sub-millisecond read latency, and batch pipeline queries check hundreds of SKUs in a single round trip. Lua scripts provide atomic stock decrements during checkout. A 5-minute TTL keeps the cache fresh while reducing database load by over 95% for read-heavy product pages.
