# How to Implement a Ring Buffer with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ring Buffer, List

Description: Implement a fixed-size ring buffer in Redis using lists with LPUSH and LTRIM to store the most recent N events with automatic overflow eviction.

---

A ring buffer (circular buffer) maintains the last N items in a fixed-size queue. Older entries are automatically discarded when new ones arrive. This is ideal for storing recent log lines, sensor readings, audit trails, or event histories without unbounded memory growth.

## Basic Ring Buffer

Use LPUSH to add to the front and LTRIM to cap the size at N:

```python
import redis
import time
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def rb_push(key: str, value: str, max_size: int):
    """Add an item and evict the oldest if buffer is full."""
    pipe = r.pipeline()
    pipe.lpush(key, value)
    pipe.ltrim(key, 0, max_size - 1)
    pipe.execute()

def rb_read(key: str) -> list:
    """Read all items, newest first."""
    return r.lrange(key, 0, -1)

def rb_read_oldest_first(key: str) -> list:
    """Read all items, oldest first."""
    return r.lrange(key, 0, -1)[::-1]

def rb_size(key: str) -> int:
    return r.llen(key)
```

## Structured Event Log

Store timestamped events as JSON:

```python
def log_event(stream_name: str, event_type: str, data: dict, max_events: int = 1000):
    event = json.dumps({
        "type": event_type,
        "data": data,
        "ts": time.time()
    })
    rb_push(f"ringbuf:{stream_name}", event, max_events)

def get_recent_events(stream_name: str, n: int = 50) -> list:
    raw = r.lrange(f"ringbuf:{stream_name}", 0, n - 1)
    return [json.loads(e) for e in raw]
```

## Sensor Reading Buffer

Keep the last 100 readings from an IoT sensor:

```python
def record_sensor(sensor_id: str, value: float, unit: str = ""):
    reading = json.dumps({"value": value, "unit": unit, "ts": time.time()})
    rb_push(f"sensor:{sensor_id}", reading, max_size=100)

def get_sensor_history(sensor_id: str) -> list:
    raw = r.lrange(f"sensor:{sensor_id}", 0, -1)
    return [json.loads(r_) for r_ in raw]

def get_latest_reading(sensor_id: str) -> dict | None:
    raw = r.lindex(f"sensor:{sensor_id}", 0)
    return json.loads(raw) if raw else None
```

## Per-User Activity Buffer

Store the last 20 actions per user for audit and UX purposes:

```python
def record_user_action(user_id: str, action: str, metadata: dict = None):
    event = json.dumps({
        "action": action,
        "meta": metadata or {},
        "ts": time.time()
    })
    rb_push(f"user:activity:{user_id}", event, max_size=20)

def get_user_recent_actions(user_id: str) -> list:
    raw = r.lrange(f"user:activity:{user_id}", 0, -1)
    return [json.loads(e) for e in raw]
```

## Thread-Safe Atomic Push

For strict capacity enforcement under concurrent writes, use a Lua script:

```lua
-- rb_push_atomic.lua
local key = KEYS[1]
local value = ARGV[1]
local max_size = tonumber(ARGV[2])
redis.call('LPUSH', key, value)
redis.call('LTRIM', key, 0, max_size - 1)
return redis.call('LLEN', key)
```

```python
rb_push_script = r.register_script("""
local key = KEYS[1]
local value = ARGV[1]
local max_size = tonumber(ARGV[2])
redis.call('LPUSH', key, value)
redis.call('LTRIM', key, 0, max_size - 1)
return redis.call('LLEN', key)
""")

def rb_push_atomic(key: str, value: str, max_size: int) -> int:
    return int(rb_push_script(keys=[key], args=[value, max_size]))
```

## Summary

Redis lists make ring buffers trivial: LPUSH adds to the front and LTRIM evicts the tail. The pattern is ideal for maintaining bounded histories of events, sensor readings, or audit logs. For concurrent writers, wrapping LPUSH and LTRIM in a Lua script ensures the size cap is always enforced atomically.
