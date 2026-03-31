# How to Use CMS.INFO in Redis to Get Count-Min Sketch Details

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Count-Min Sketch, Probabilistic Data Structure, Monitoring

Description: Learn how to use CMS.INFO in Redis to inspect Count-Min Sketch metadata including width, depth, and total event count for capacity planning and debugging.

---

`CMS.INFO` returns the internal metadata for a Count-Min Sketch (CMS) stored in Redis. It provides the sketch's width, depth, and total element count - the three key parameters you need to understand the sketch's accuracy, memory usage, and current utilization. This command is essential for monitoring and debugging probabilistic frequency tracking.

## Basic Usage

```bash
# Create and populate a CMS sketch
127.0.0.1:6379> CMS.INITBYDIM events:sketch 2000 10
OK
127.0.0.1:6379> CMS.INCRBY events:sketch "click" 50 "view" 200 "purchase" 10
1) (integer) 50
2) (integer) 200
3) (integer) 10

# Inspect the sketch
127.0.0.1:6379> CMS.INFO events:sketch
1) width
2) (integer) 2000
3) depth
4) (integer) 10
5) count
6) (integer) 260
```

## Fields Explained

| Field | Description |
|---|---|
| width | Number of counters per row (affects accuracy) |
| depth | Number of hash rows (affects confidence) |
| count | Total number of events counted across all items |

The `count` field is the sum of all increments, not the number of distinct items. It helps you understand the sketch's utilization level.

## Python Example: Parsing CMS.INFO

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_cms_info(key: str) -> dict:
    """Fetch and parse CMS.INFO into a dictionary."""
    raw = r.execute_command("CMS.INFO", key)
    return {raw[i]: raw[i + 1] for i in range(0, len(raw), 2)}

def describe_cms(key: str) -> None:
    info = get_cms_info(key)
    width = info["width"]
    depth = info["depth"]
    count = info["count"]

    # Accuracy bound: error <= (e / width) * total_count
    import math
    error_bound = (math.e / width) * count if count > 0 else 0

    print(f"Sketch: {key}")
    print(f"  Width:          {width}")
    print(f"  Depth:          {depth}")
    print(f"  Total events:   {count:,}")
    print(f"  Max error bound: ~{error_bound:,.0f} counts per item")

# Setup
r.execute_command("CMS.INITBYDIM", "api:traffic", 2000, 7)
r.execute_command("CMS.INCRBY", "api:traffic",
                  "/users", 5000,
                  "/orders", 1500,
                  "/products", 8000)

describe_cms("api:traffic")
```

## Monitoring Sketch Saturation

While a CMS doesn't "fill up" like a Bloom filter, a very high count relative to width can increase effective error rates. You can track this:

```python
import math
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_accuracy_report(key: str) -> None:
    info = get_cms_info(key)
    width = info["width"]
    depth = info["depth"]
    count = info["count"]

    # Theoretical error bound as percentage of total count
    if count > 0:
        error_frac = math.e / width
        print(f"Theoretical error: up to {error_frac*100:.3f}% of total count")
        print(f"Absolute max error per item: ~{error_frac * count:,.0f}")
    else:
        print("Sketch is empty - no events counted yet")

get_accuracy_report("api:traffic")
```

## Checking Sketch Existence Before Operations

```python
def cms_exists(key: str) -> bool:
    try:
        r.execute_command("CMS.INFO", key)
        return True
    except redis.ResponseError:
        return False

if not cms_exists("new:sketch"):
    r.execute_command("CMS.INITBYDIM", "new:sketch", 1000, 5)
    print("Sketch created")
else:
    print("Sketch already exists")
```

## Comparing Multiple Sketches

```python
sketches = ["api:traffic", "user:events", "product:views"]

for key in sketches:
    try:
        info = get_cms_info(key)
        print(f"{key}: width={info['width']}, depth={info['depth']}, "
              f"count={info['count']:,}")
    except redis.ResponseError:
        print(f"{key}: not found")
```

## Summary

`CMS.INFO` provides the essential metadata for a Count-Min Sketch: width, depth, and total event count. Use it to verify sketch dimensions after creation, monitor total event volume, calculate theoretical error bounds, and build dashboards that track sketch health over time. It's the go-to command for understanding the current state of any CMS-based frequency tracking system in Redis.
