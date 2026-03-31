# How to Implement IoT Device Heartbeat Monitoring with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, IoT, Heartbeat, TTL, Monitoring

Description: Monitor IoT device connectivity by using Redis TTL keys as heartbeat timers that automatically expire when devices go silent, triggering offline detection without polling loops.

---

Device heartbeat monitoring tells you which devices are alive. Instead of polling each device, you can invert the model: devices write to Redis on every heartbeat and a missing key means a missed heartbeat. Redis TTL handles the timer automatically.

## Heartbeat Key Design

Each device writes a heartbeat key with a TTL equal to the expected interval plus tolerance:

```python
import redis
r = redis.Redis()

HEARTBEAT_INTERVAL = 30   # device sends every 30s
HEARTBEAT_TTL = 90        # 3x interval for tolerance

def record_heartbeat(device_id):
    r.set(f"heartbeat:{device_id}", 1, ex=HEARTBEAT_TTL)
```

If the device misses three intervals, the key expires and the device is considered offline.

## Checking Device Status

Check a single device:

```python
def is_device_online(device_id):
    return r.exists(f"heartbeat:{device_id}") == 1
```

Get the remaining TTL to estimate time since last heartbeat:

```python
def seconds_since_heartbeat(device_id):
    ttl = r.ttl(f"heartbeat:{device_id}")
    if ttl == -2:
        return None  # Key does not exist - device offline
    return HEARTBEAT_TTL - ttl
```

## Keyspace Notifications for Offline Detection

Enable expired key notifications in Redis config:

```bash
redis-cli CONFIG SET notify-keyspace-events "Ex"
```

Subscribe to expiry events to detect devices going offline automatically:

```python
def watch_device_expirations():
    pubsub = r.pubsub()
    pubsub.psubscribe("__keyevent@0__:expired")
    for message in pubsub.listen():
        if message["type"] != "pmessage":
            continue
        key = message["data"].decode()
        if key.startswith("heartbeat:"):
            device_id = key.split(":", 1)[1]
            on_device_offline(device_id)

def on_device_offline(device_id):
    r.sadd("devices:offline", device_id)
    r.srem("devices:online", device_id)
    trigger_offline_alert(device_id)
```

## Fleet Health Summary

Maintain separate sets for online and offline devices:

```python
def record_heartbeat_with_tracking(device_id):
    was_offline = r.sismember("devices:offline", device_id)
    r.set(f"heartbeat:{device_id}", 1, ex=HEARTBEAT_TTL)
    r.sadd("devices:online", device_id)
    if was_offline:
        r.srem("devices:offline", device_id)
        on_device_recovered(device_id)
```

Query fleet status:

```bash
SCARD devices:online
SCARD devices:offline
```

## Heartbeat Metadata

Store additional context with each heartbeat:

```python
import json
import time

def record_rich_heartbeat(device_id, metadata: dict):
    payload = {"ts": time.time(), **metadata}
    r.set(f"heartbeat:{device_id}", json.dumps(payload), ex=HEARTBEAT_TTL)
```

Retrieve diagnostics on last contact:

```python
def get_last_heartbeat(device_id):
    data = r.get(f"heartbeat:{device_id}")
    return json.loads(data) if data else None
```

## Summary

Redis TTL keys make heartbeat monitoring elegant - devices write their presence, and Redis automatically expires the key when they go silent. Keyspace notifications push offline events in real time instead of requiring polling, and set-based online/offline tracking gives instant fleet-wide health visibility.
