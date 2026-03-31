# How to Build a Warehouse Location Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Warehouse, Cache

Description: Cache warehouse bin locations and SKU-to-location mappings in Redis to speed up pick-and-pack operations and reduce database load.

---

In a warehouse, pickers need to know exactly where to find each item. Querying a relational database for every pick request adds latency and creates hot spots. Redis gives you a sub-millisecond location lookup cache.

## Storing Bin Locations

Map each SKU to its bin location and store metadata in a hash:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def set_bin_location(sku: str, bin_id: str, zone: str, aisle: str, shelf: str):
    pipe = r.pipeline()
    # Direct SKU-to-bin mapping for fast lookup
    pipe.set(f"location:sku:{sku}", bin_id, ex=3600)  # 1-hour TTL

    # Full location details in a hash
    pipe.hset(f"bin:{bin_id}", mapping={
        "zone": zone,
        "aisle": aisle,
        "shelf": shelf,
        "sku": sku
    })

    # Zone index (set of bins in each zone)
    pipe.sadd(f"zone:{zone}:bins", bin_id)
    pipe.execute()

def get_pick_location(sku: str) -> dict | None:
    bin_id = r.get(f"location:sku:{sku}")
    if not bin_id:
        return None  # Cache miss - fetch from DB
    details = r.hgetall(f"bin:{bin_id}")
    return {"bin_id": bin_id, **details}
```

## Bulk Location Lookup for Pick Lists

A picker usually receives a list of SKUs. Fetch all locations in a single pipeline:

```python
def get_pick_list_locations(skus: list[str]) -> dict:
    pipe = r.pipeline(transaction=False)
    for sku in skus:
        pipe.get(f"location:sku:{sku}")
    bin_ids = pipe.execute()

    # Fetch bin details for any that had a cache hit
    pipe2 = r.pipeline(transaction=False)
    for bin_id in bin_ids:
        if bin_id:
            pipe2.hgetall(f"bin:{bin_id}")
        else:
            pipe2.hgetall("_nonexistent_")  # placeholder
    details = pipe2.execute()

    result = {}
    for i, sku in enumerate(skus):
        if bin_ids[i]:
            result[sku] = {"bin_id": bin_ids[i], **details[i]}
        else:
            result[sku] = None  # needs DB fallback
    return result
```

## Updating Location After Putaway

When a SKU moves to a new bin:

```python
def move_sku_to_bin(sku: str, new_bin_id: str, zone: str, aisle: str, shelf: str):
    old_bin_id = r.get(f"location:sku:{sku}")

    pipe = r.pipeline()
    # Update SKU location pointer
    pipe.set(f"location:sku:{sku}", new_bin_id, ex=3600)

    # Update new bin hash
    pipe.hset(f"bin:{new_bin_id}", mapping={
        "zone": zone, "aisle": aisle, "shelf": shelf, "sku": sku
    })

    # Remove SKU from old bin if known
    if old_bin_id:
        pipe.hdel(f"bin:{old_bin_id}", "sku")
        old_zone = r.hget(f"bin:{old_bin_id}", "zone")
        if old_zone:
            pipe.srem(f"zone:{old_zone}:bins", old_bin_id)

    pipe.sadd(f"zone:{zone}:bins", new_bin_id)
    pipe.execute()
```

## Nearest Bin by Zone

To optimize picker routes, get all bins in a zone:

```python
def get_bins_in_zone(zone: str) -> list:
    bin_ids = r.smembers(f"zone:{zone}:bins")
    pipe = r.pipeline(transaction=False)
    for bin_id in bin_ids:
        pipe.hgetall(f"bin:{bin_id}")
    details = pipe.execute()
    return [{"bin_id": bid, **d} for bid, d in zip(bin_ids, details) if d]
```

## Cache Invalidation on Stock Changes

When inventory is depleted or restocked:

```python
def invalidate_location(sku: str):
    r.delete(f"location:sku:{sku}")
    # On next lookup, the app will fetch from DB and re-cache
```

## Summary

A Redis location cache eliminates repeated database queries for high-frequency SKU lookups in warehouse operations. Pipeline-based bulk fetches minimize round-trips for pick lists, and short TTLs ensure the cache stays consistent with the warehouse management system without manual invalidation overhead.
