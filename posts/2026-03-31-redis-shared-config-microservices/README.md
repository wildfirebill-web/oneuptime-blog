# How to Implement Shared Configuration with Redis in Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Configuration, Microservice, Pub/Sub, Feature Flag

Description: Use Redis as a shared configuration store for microservices, with Pub/Sub-based hot reloading so services pick up config changes without restarting.

---

Microservices often need shared configuration: feature flags, rate limit values, service URLs, or timeouts. Environment variables require restarts to change. A Redis-backed config store allows hot reloading - services pick up changes within seconds.

## Config Storage Structure

Store configuration as Redis Hashes grouped by namespace:

```bash
# Global feature flags
HSET config:features \
  dark_mode "true" \
  new_checkout "false" \
  max_cart_size "50"

# Service-specific config
HSET config:payment \
  timeout_ms "3000" \
  retry_count "3" \
  currency "USD"
```

## Reading Config in a Service

```python
import redis
import json

r = redis.Redis(host="redis", port=6379, decode_responses=True)

class ConfigStore:
    def __init__(self, namespace: str):
        self.namespace = namespace
        self._cache = {}
        self._load()

    def _load(self):
        self._cache = r.hgetall(f"config:{self.namespace}")

    def get(self, key: str, default=None):
        return self._cache.get(key, default)

    def get_int(self, key: str, default: int = 0) -> int:
        return int(self._cache.get(key, default))

    def get_bool(self, key: str, default: bool = False) -> bool:
        val = self._cache.get(key, str(default)).lower()
        return val in ("true", "1", "yes")

config = ConfigStore("payment")
timeout = config.get_int("timeout_ms", 5000)
```

## Hot Reload via Pub/Sub

Publish a reload signal when config changes:

```python
def update_config(namespace: str, key: str, value: str):
    r.hset(f"config:{namespace}", key, value)
    r.publish(f"config:reload", namespace)
```

Services subscribe and reload their local cache:

```python
import threading

class HotReloadingConfig(ConfigStore):
    def __init__(self, namespace: str):
        super().__init__(namespace)
        self._start_listener()

    def _start_listener(self):
        def listen():
            sub = r.pubsub()
            sub.subscribe("config:reload")
            for msg in sub.listen():
                if msg["type"] == "message" and msg["data"] == self.namespace:
                    self._load()
                    print(f"Config reloaded for {self.namespace}")

        thread = threading.Thread(target=listen, daemon=True)
        thread.start()

config = HotReloadingConfig("payment")
```

## Feature Flags

```python
def is_feature_enabled(flag: str, user_id: int = None) -> bool:
    value = r.hget("config:features", flag)
    if value is None:
        return False

    # Percentage rollout: "rollout:10" = enabled for 10% of users
    if value.startswith("rollout:") and user_id is not None:
        pct = int(value.split(":")[1])
        return (user_id % 100) < pct

    return value.lower() == "true"
```

## Versioning and Audit Trail

Track config changes with a log:

```python
import time

def update_config_versioned(namespace: str, key: str, value: str, author: str):
    r.hset(f"config:{namespace}", key, value)

    # Append to audit log
    entry = json.dumps({
        "namespace": namespace,
        "key": key,
        "value": value,
        "author": author,
        "timestamp": int(time.time())
    })
    r.lpush("config:audit_log", entry)
    r.ltrim("config:audit_log", 0, 999)  # keep last 1000 changes

    r.publish("config:reload", namespace)
```

## Bootstrapping from a File

On startup, load defaults from a file if Redis is empty:

```python
import yaml

def bootstrap_config(namespace: str, config_file: str):
    if r.exists(f"config:{namespace}"):
        return  # already initialized

    with open(config_file) as f:
        defaults = yaml.safe_load(f)

    r.hset(f"config:{namespace}", mapping=defaults)
```

## Summary

Redis shared configuration gives microservices a single source of truth for runtime settings without the overhead of a full config server. Pub/Sub hot reload lets services update their local cache within milliseconds of a config change. Feature flags with percentage-based rollouts can be layered on top using the same hash structure.
