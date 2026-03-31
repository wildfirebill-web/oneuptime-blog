# How to Use CF.INFO in Redis to Get Cuckoo Filter Details

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cuckoo Filter, Probabilistic Data Structure, Monitoring

Description: Learn how to use CF.INFO in Redis to inspect Cuckoo filter metadata including size, item count, bucket configuration, and expansion status.

---

`CF.INFO` returns detailed metadata about a Cuckoo filter stored in Redis. It's the primary way to inspect a filter's current state - how full it is, how many items have been inserted or deleted, and what its structural parameters are. This is essential for capacity planning and debugging.

## Basic Usage

```bash
127.0.0.1:6379> CF.RESERVE product:ids 100000 BUCKETSIZE 4
OK
127.0.0.1:6379> CF.ADD product:ids "SKU-001"
(integer) 1
127.0.0.1:6379> CF.ADD product:ids "SKU-002"
(integer) 1
127.0.0.1:6379> CF.INFO product:ids
 1) Size
 2) (integer) 65536
 3) Number of buckets
 4) (integer) 16384
 5) Number of filter
 6) (integer) 1
 7) Number of items inserted
 8) (integer) 2
 9) Number of items deleted
10) (integer) 0
11) Bucket size
12) (integer) 4
13) Expansion rate
14) (integer) 1
15) Max iterations
16) (integer) 20
```

## Fields Explained

| Field | Description |
|---|---|
| Size | Total size in bytes |
| Number of buckets | Total buckets across all sub-filters |
| Number of filter | How many sub-filters exist (grows with expansion) |
| Number of items inserted | Total items added (includes re-inserts) |
| Number of items deleted | Total items deleted |
| Bucket size | Items per bucket |
| Expansion rate | Growth multiplier when filter is full |
| Max iterations | Max swap attempts during insertion |

## Python Example: Monitoring Filter Health

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_cf_info(key: str) -> dict:
    """Fetch and parse CF.INFO output into a dictionary."""
    raw = r.execute_command("CF.INFO", key)
    # CF.INFO returns alternating key/value list
    return {raw[i]: raw[i + 1] for i in range(0, len(raw), 2)}

def report_filter_status(key: str) -> None:
    info = get_cf_info(key)
    inserted = info["Number of items inserted"]
    deleted = info["Number of items deleted"]
    active = inserted - deleted
    size_bytes = info["Size"]
    num_filters = info["Number of filter"]

    print(f"Filter: {key}")
    print(f"  Active items:  {active:,}")
    print(f"  Inserted:      {inserted:,}")
    print(f"  Deleted:       {deleted:,}")
    print(f"  Memory usage:  {size_bytes:,} bytes")
    print(f"  Sub-filters:   {num_filters} (1 = original, >1 = expanded)")
    print(f"  Bucket size:   {info['Bucket size']}")

# Setup
r.execute_command("CF.RESERVE", "session:active", 50000, "BUCKETSIZE", 4)
for i in range(100):
    r.execute_command("CF.ADD", "session:active", f"session:{i}")
r.execute_command("CF.DEL", "session:active", "session:5")

report_filter_status("session:active")
```

## Detecting Expansion

When a Cuckoo filter fills up and `EXPANSION` is not 0, Redis creates additional sub-filters. You can detect this:

```python
def is_expanded(key: str) -> bool:
    info = get_cf_info(key)
    return info["Number of filter"] > 1

def load_factor(key: str) -> float:
    info = get_cf_info(key)
    inserted = info["Number of items inserted"]
    deleted = info["Number of items deleted"]
    buckets = info["Number of buckets"]
    bucket_size = info["Bucket size"]
    capacity = buckets * bucket_size
    return (inserted - deleted) / capacity * 100

print(f"Expanded: {is_expanded('session:active')}")
print(f"Load: {load_factor('session:active'):.1f}%")
```

## Alerting on Filter Saturation

```python
def check_filter_alerts(key: str, warn_pct: float = 80.0) -> None:
    load = load_factor(key)
    if load > warn_pct:
        print(f"WARNING: Filter '{key}' is {load:.1f}% full - consider re-provisioning")
    else:
        print(f"OK: Filter '{key}' load is {load:.1f}%")

check_filter_alerts("session:active")
```

## Summary

`CF.INFO` gives you a complete snapshot of a Cuckoo filter's internal state, including item counts, memory usage, bucket configuration, and expansion status. Use it for capacity planning, debugging unexpected behavior, and building monitoring dashboards that alert when filters approach saturation.
