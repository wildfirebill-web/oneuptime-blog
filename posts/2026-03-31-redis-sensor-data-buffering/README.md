# How to Implement Sensor Data Buffering with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, IoT, Sensor, Stream, Buffer

Description: Use Redis lists and Streams to buffer high-frequency IoT sensor data before batch-writing to a time-series database, reducing write pressure and handling bursts gracefully.

---

IoT sensors can emit thousands of readings per second. Writing each reading directly to a time-series database creates high write pressure and connection overhead. Redis acts as a fast intermediate buffer that absorbs bursts and delivers batches to downstream storage.

## Simple List Buffer

For basic buffering, push readings onto a Redis list:

```python
import redis
import json
import time

r = redis.Redis()

def ingest_reading(device_id, metric, value):
    reading = json.dumps({
        "device_id": device_id,
        "metric": metric,
        "value": value,
        "ts": time.time()
    })
    r.rpush(f"buffer:{device_id}", reading)
```

## Batch Drain

A background worker drains the buffer in batches:

```python
def drain_buffer(device_id, batch_size=100):
    pipe = r.pipeline()
    pipe.lrange(f"buffer:{device_id}", 0, batch_size - 1)
    pipe.ltrim(f"buffer:{device_id}", batch_size, -1)
    results, _ = pipe.execute()
    readings = [json.loads(r) for r in results]
    if readings:
        write_to_timeseries(readings)
    return len(readings)
```

## Using Redis Streams for Durability

Streams offer persistence and consumer group support, which is better for production:

```bash
XADD sensor:temperature:d-001 * value 23.4 unit celsius
XADD sensor:temperature:d-001 * value 23.5 unit celsius
```

```python
def ingest_stream(device_id, metric, value, unit):
    stream_key = f"sensor:{metric}:{device_id}"
    r.xadd(stream_key, {"value": value, "unit": unit})
```

## Stream-Based Consumer

Read and process in batches with consumer groups:

```python
def setup_consumer_group(stream_key):
    try:
        r.xgroup_create(stream_key, "storage-workers", id="0", mkstream=True)
    except redis.exceptions.ResponseError:
        pass  # group already exists

def consume_and_store(stream_key, consumer_name, count=200):
    messages = r.xreadgroup(
        "storage-workers", consumer_name,
        {stream_key: ">"},
        count=count, block=1000
    )
    for stream, entries in (messages or []):
        ids = []
        batch = []
        for msg_id, fields in entries:
            batch.append(fields)
            ids.append(msg_id)
        if batch:
            write_to_timeseries(batch)
            r.xack(stream_key, "storage-workers", *ids)
```

## Capping Stream Length

Prevent unbounded growth by capping the stream:

```bash
XADD sensor:temperature:d-001 MAXLEN ~ 10000 * value 23.6 unit celsius
```

The `~` flag allows Redis to trim approximately rather than exactly, improving performance.

## Monitoring Buffer Depth

Alert when the buffer grows too large:

```python
def check_buffer_health(device_id, metric, warn_threshold=5000):
    depth = r.xlen(f"sensor:{metric}:{device_id}")
    if depth > warn_threshold:
        alert(f"Buffer too deep for {device_id}: {depth} entries")
    return depth
```

## Summary

Redis lists provide simple, fast buffering for sensor data while Redis Streams add durability, consumer groups, and acknowledgment semantics for production workloads. Capping stream length and monitoring buffer depth ensure the system stays healthy under sustained high ingestion rates.
