# How to Build a Dynamic Configuration System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Configuration, Dynamic Config, Backend, Distributed System

Description: Build a dynamic configuration system using Redis hashes and keyspace notifications so services pick up config changes instantly without restarting.

---

Static config files require service restarts to apply changes. A Redis-backed dynamic configuration system lets you update values at runtime and have all running instances pick up the change within seconds.

## Storing Configuration as Hashes

Group related config values under a hash key per service or per environment:

```bash
# Set config for the payments service
HSET config:payments max_retry_count 3
HSET config:payments timeout_ms 5000
HSET config:payments enabled_currencies "USD,EUR,GBP"
```

## Reading Config in Your Application

Load all config values at startup and cache them locally for fast access:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

class DynamicConfig:
    def __init__(self, service: str):
        self.service = service
        self._cache = {}
        self._load()

    def _load(self):
        self._cache = r.hgetall(f"config:{self.service}")

    def get(self, key: str, default=None):
        return self._cache.get(key, default)
```

## Live Updates via Keyspace Notifications

Enable keyspace notifications in Redis and subscribe to changes on the config hash:

```bash
# In redis.conf or via CONFIG SET
CONFIG SET notify-keyspace-events KEh
```

```python
import threading

class DynamicConfig:
    def __init__(self, service: str):
        self.service = service
        self._cache = {}
        self._load()
        threading.Thread(target=self._watch, daemon=True).start()

    def _load(self):
        self._cache = r.hgetall(f"config:{self.service}")

    def _watch(self):
        sub = r.pubsub()
        channel = f"__keyevent@0__:hset"
        sub.subscribe(channel)
        for msg in sub.listen():
            if msg["type"] == "message":
                key = msg["data"]
                if key == f"config:{self.service}":
                    self._load()

    def get(self, key: str, default=None):
        return self._cache.get(key, default)
```

## Updating Config Safely

Update a single value and let all subscribers reload:

```python
def update_config(service: str, key: str, value: str):
    r.hset(f"config:{service}", key, value)
    # Keyspace notification triggers auto-reload in all listeners
```

## Config Change Audit Log

Track who changed what and when using a Redis stream:

```python
import time

def update_config_with_audit(service: str, key: str, value: str, changed_by: str):
    r.hset(f"config:{service}", key, value)
    r.xadd(f"config:audit:{service}", {
        "key": key,
        "value": value,
        "changed_by": changed_by,
        "timestamp": str(time.time()),
    })
```

## Summary

Redis hashes provide a flat, fast key-value store for service configuration. Keyspace notifications push changes to all subscribers instantly, eliminating restarts. Adding a stream-based audit log gives you a full change history that you can query or replay to reconstruct any historical config state.

