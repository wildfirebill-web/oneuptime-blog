# How to Implement Multi-Dimensional Counting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Analytics, Multi-Dimensional, Hash

Description: Count events across multiple dimensions simultaneously in Redis using composite keys and hash fields for slice-and-dice analytics.

---

Single-dimensional counters tell you total requests. Multi-dimensional counters tell you requests by region, by endpoint, by status code, and by all combinations at once. Redis composite keys and hashes make this efficient without a dedicated OLAP system.

## Composite Key Strategy

Encode dimensions directly in the key name to support any combination of slice queries:

```python
import redis
import time
import itertools

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def record_request(region: str, endpoint: str, status: str, method: str):
    now = time.strftime("%Y%m%d%H", time.gmtime())
    dimensions = {
        "region": region,
        "endpoint": endpoint,
        "status": status,
        "method": method,
    }

    pipe = r.pipeline()
    # Total counter
    pipe.incr(f"req:{now}:total")

    # Single-dimension counters
    for dim, val in dimensions.items():
        key = f"req:{now}:{dim}:{val}"
        pipe.incr(key)
        pipe.expire(key, 7 * 86400)

    # Two-dimension combinations
    for (d1, v1), (d2, v2) in itertools.combinations(dimensions.items(), 2):
        key = f"req:{now}:{d1}:{v1}:{d2}:{v2}"
        pipe.incr(key)
        pipe.expire(key, 7 * 86400)

    pipe.execute()
```

## Hash-Based Multi-Dimensional Counter

For a smaller, fixed set of dimensions, use a single hash per time bucket:

```python
def record_event_hash(service: str, status: str, region: str):
    hour = time.strftime("%Y%m%d%H", time.gmtime())
    key = f"events:{hour}"
    pipe = r.pipeline()
    pipe.hincrby(key, f"svc:{service}", 1)
    pipe.hincrby(key, f"status:{status}", 1)
    pipe.hincrby(key, f"region:{region}", 1)
    pipe.hincrby(key, f"svc:{service}:status:{status}", 1)
    pipe.hincrby(key, "total", 1)
    pipe.expire(key, 7 * 86400)
    pipe.execute()
```

## Querying a Dimension Slice

Read all values for a dimension prefix:

```python
def get_dimension_slice(dimension: str, hour: str = None) -> dict:
    hour = hour or time.strftime("%Y%m%d%H", time.gmtime())
    key = f"events:{hour}"
    all_fields = r.hgetall(key)
    prefix = f"{dimension}:"
    return {
        k[len(prefix):]: int(v)
        for k, v in all_fields.items()
        if k.startswith(prefix)
    }
```

## Top N by Dimension Using Sorted Sets

Use sorted sets to maintain a ranked view per dimension:

```python
def record_with_ranking(endpoint: str, region: str):
    hour = time.strftime("%Y%m%d%H", time.gmtime())
    pipe = r.pipeline()
    pipe.zincrby(f"top:endpoints:{hour}", 1, endpoint)
    pipe.zincrby(f"top:regions:{hour}", 1, region)
    pipe.expire(f"top:endpoints:{hour}", 7 * 86400)
    pipe.expire(f"top:regions:{hour}", 7 * 86400)
    pipe.execute()

def get_top_n(dimension: str, n: int = 10, hour: str = None) -> list:
    hour = hour or time.strftime("%Y%m%d%H", time.gmtime())
    key = f"top:{dimension}:{hour}"
    results = r.zrevrangebyscore(key, "+inf", "-inf", start=0, num=n, withscores=True)
    return [{"value": v, "count": int(s)} for v, s in results]
```

## Aggregating Across Hours

Sum a dimension across multiple hours:

```python
def get_dimension_totals(dimension: str, value: str, hours: int = 24) -> int:
    now = time.time()
    total = 0
    for i in range(hours):
        ts = now - i * 3600
        hour = time.strftime("%Y%m%d%H", time.gmtime(ts))
        v = r.hget(f"events:{hour}", f"{dimension}:{value}")
        total += int(v or 0)
    return total
```

## Summary

Multi-dimensional counting in Redis uses composite key names and hash fields to support slice-and-dice queries without a query engine. Writing to multiple dimension keys in a single pipeline is fast, and sorted sets provide ranked views per dimension. This pattern works for request analytics, error breakdowns, and user segmentation at high event rates.
