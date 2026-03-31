# How to Use Redis for Edge Computing Use Cases

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Edge Computing, IoT

Description: Learn how to deploy Redis at the edge for IoT data buffering, local caching, and real-time aggregation with examples for constrained and distributed environments.

---

Edge computing moves processing closer to data sources - IoT devices, CDN nodes, factory floors, and regional servers. Redis is an excellent edge data store because it is lightweight, fast, and runs on minimal hardware. A single Redis instance can handle tens of thousands of operations per second on a Raspberry Pi.

## Why Redis at the Edge

- Low memory footprint (under 10 MB idle)
- Sub-millisecond reads and writes
- Works offline without a central server
- Supports TTL-based automatic data expiration
- Sync to cloud Redis when connectivity resumes

## Buffer IoT Sensor Readings

```python
import redis
import time
import random

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def buffer_sensor_reading(device_id: str, temperature: float, humidity: float):
    key = f"sensor:{device_id}:readings"
    reading = f"{time.time()},{temperature},{humidity}"
    r.rpush(key, reading)
    # Keep last 1000 readings
    r.ltrim(key, -1000, -1)

# Simulate sensor data
for i in range(10):
    buffer_sensor_reading("device-001", round(22 + random.uniform(-2, 2), 2), round(55 + random.uniform(-5, 5), 2))

print(r.llen("sensor:device-001:readings"), "readings buffered")
```

## Aggregate in Real Time

Compute running averages without storing all raw data:

```python
def update_stats(r, device_id: str, temperature: float):
    pipe = r.pipeline()
    pipe.incr(f"stats:{device_id}:count")
    pipe.incrbyfloat(f"stats:{device_id}:temp_sum", temperature)
    count, temp_sum = pipe.execute()
    avg = float(temp_sum) / int(count)
    r.set(f"stats:{device_id}:avg_temp", round(avg, 3))

update_stats(r, "device-001", 23.4)
print(r.get("stats:device-001:avg_temp"))
```

## Track Device Presence with TTL

Use TTL-based keys as a heartbeat mechanism:

```python
def heartbeat(r, device_id: str, ttl_seconds: int = 30):
    r.setex(f"device:online:{device_id}", ttl_seconds, "1")

def is_online(r, device_id: str) -> bool:
    return r.exists(f"device:online:{device_id}") == 1

heartbeat(r, "device-001")
print(is_online(r, "device-001"))  # True
# After 30 seconds of no heartbeats, the key expires
```

## Queue Commands for Offline Sync

When the edge device is temporarily disconnected from the cloud, queue data for later sync:

```python
def queue_for_sync(r, payload: dict):
    import json
    r.rpush("sync:queue", json.dumps(payload))

def flush_to_cloud(r, cloud_r):
    while True:
        item = r.lpop("sync:queue")
        if not item:
            break
        import json
        data = json.loads(item)
        cloud_r.xadd("cloud:sensor:data", data)
        print(f"Synced: {data}")

# Edge device queues data
queue_for_sync(r, {"device": "device-001", "temp": 23.4, "ts": time.time()})
```

## Publish Alerts Locally

Edge devices can publish alerts to local subscribers without involving the cloud:

```python
# Publisher on edge
def publish_alert(r, device_id: str, message: str):
    r.publish(f"alerts:{device_id}", message)

# Subscriber on edge (runs in separate thread/process)
def listen_for_alerts(r, device_id: str):
    pubsub = r.pubsub()
    pubsub.subscribe(f"alerts:{device_id}")
    for message in pubsub.listen():
        if message['type'] == 'message':
            print(f"Alert from {device_id}: {message['data']}")
```

## Run Redis with Minimal Config on Edge Hardware

```text
# /etc/redis/redis-edge.conf
maxmemory 64mb
maxmemory-policy allkeys-lru
save ""
appendonly no
bind 127.0.0.1
```

```bash
redis-server /etc/redis/redis-edge.conf --daemonize yes
```

## Summary

Redis works well at the edge for buffering sensor data, computing local aggregations, tracking device presence with TTLs, and queuing payloads for cloud sync. Its low resource requirements and fast in-memory operations make it practical for Raspberry Pis, industrial gateways, and regional CDN nodes running constrained workloads.
