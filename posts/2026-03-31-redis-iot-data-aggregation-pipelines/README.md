# How to Build IoT Data Aggregation Pipelines with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, IoT, Aggregation, Stream, Pipeline

Description: Build multi-stage IoT data aggregation pipelines using Redis Streams and sorted sets to compute rolling averages, per-device summaries, and fleet-wide statistics in real time.

---

Raw IoT sensor readings are rarely useful on their own. You need aggregations: per-device averages, hourly summaries, fleet-wide anomaly counts. Redis Streams and atomic operations let you compute these in a pipeline without a separate analytics engine.

## Ingestion Stage

Ingest raw readings into a stream:

```python
import redis
import time
r = redis.Redis()

def ingest(device_id, metric, value):
    r.xadd(f"raw:{metric}", {"device_id": device_id, "value": value, "ts": time.time()},
           maxlen=100000, approximate=True)
```

## Rolling Average with Sorted Set

Compute a per-device rolling average over the last 60 readings:

```python
def update_rolling_avg(device_id, metric, value):
    key = f"rolling:{metric}:{device_id}"
    r.zadd(key, {f"{time.time()}:{value}": time.time()})
    # Keep only last 60 readings
    r.zremrangebyrank(key, 0, -61)

def get_rolling_avg(device_id, metric):
    key = f"rolling:{metric}:{device_id}"
    members = r.zrange(key, 0, -1)
    if not members:
        return None
    values = [float(m.decode().split(":")[1]) for m in members]
    return sum(values) / len(values)
```

## Minute-Level Aggregation

Compute per-minute stats using a consumer group on the raw stream:

```python
from collections import defaultdict

def aggregate_minute(stream_key, consumer_name):
    messages = r.xreadgroup("agg-workers", consumer_name,
                             {stream_key: ">"}, count=500, block=1000)
    buckets = defaultdict(list)
    ids = []
    for _, entries in (messages or []):
        for msg_id, fields in entries:
            device_id = fields[b"device_id"].decode()
            value = float(fields[b"value"])
            minute = int(float(fields[b"ts"]) // 60) * 60
            buckets[(device_id, minute)].append(value)
            ids.append(msg_id)
    for (device_id, minute), values in buckets.items():
        store_minute_agg(device_id, stream_key, minute, values)
    if ids:
        r.xack(stream_key, "agg-workers", *ids)
```

## Storing Minute Aggregations

Write computed stats as hashes:

```python
def store_minute_agg(device_id, metric, minute, values):
    key = f"agg:minute:{metric}:{device_id}:{minute}"
    r.hset(key, mapping={
        "min": min(values),
        "max": max(values),
        "avg": sum(values) / len(values),
        "count": len(values)
    })
    r.expire(key, 86400 * 7)  # Retain for 7 days
```

## Fleet-Wide Counters

Maintain fleet-wide totals using Redis counters:

```python
def record_fleet_stats(metric, value):
    hour = int(time.time() // 3600) * 3600
    r.incrbyfloat(f"fleet:{metric}:total:{hour}", value)
    r.incr(f"fleet:{metric}:count:{hour}")
    r.expire(f"fleet:{metric}:total:{hour}", 86400 * 3)
    r.expire(f"fleet:{metric}:count:{hour}", 86400 * 3)
```

## Querying Hourly Summary

Calculate the fleet average for the past hour:

```python
def fleet_hourly_avg(metric):
    hour = int(time.time() // 3600) * 3600
    total = float(r.get(f"fleet:{metric}:total:{hour}") or 0)
    count = int(r.get(f"fleet:{metric}:count:{hour}") or 0)
    return total / count if count else None
```

## Summary

Redis Streams and atomic operations form an efficient multi-stage IoT aggregation pipeline: raw ingestion into streams, per-device rolling averages with sorted sets, minute-level stats in hashes, and fleet-wide counters for dashboards. This approach computes real-time insights without requiring a dedicated stream processing framework.
