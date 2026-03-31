# How to Build a Remote Config System for Mobile Apps with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Remote Config, Mobile, Configuration, Backend

Description: Build a Redis-backed remote config API for mobile apps that delivers per-platform and per-version config bundles with versioning and fast cache invalidation.

---

Mobile apps cannot be hot-patched like web apps. A remote config system lets your backend control feature flags, UI strings, and API endpoints without an app store release. Redis is the ideal backing store for fast config delivery.

## Storing Config Bundles by Platform and Version

Organize configs using a compound key that identifies platform and minimum app version:

```bash
HSET config:ios:v3 feature_new_onboarding 1
HSET config:ios:v3 max_items_per_page 20
HSET config:ios:v3 api_base_url "https://api.example.com/v3"

HSET config:android:v3 feature_new_onboarding 1
HSET config:android:v3 max_items_per_page 25
```

## Config Resolution Endpoint

Your API resolves the best-matching config bundle for an app's platform and version:

```python
import redis
from flask import Flask, jsonify, request

app = Flask(__name__)
r = redis.Redis(host="localhost", port=6379, decode_responses=True)

SUPPORTED_VERSIONS = ["v4", "v3", "v2", "v1"]

@app.get("/remote-config")
def get_remote_config():
    platform = request.args.get("platform", "ios")
    app_version = request.args.get("version", "v1")

    # Walk from the given version down to find the closest config
    versions = SUPPORTED_VERSIONS[SUPPORTED_VERSIONS.index(app_version):]
    for v in versions:
        config = r.hgetall(f"config:{platform}:{v}")
        if config:
            return jsonify({"version": v, "config": config})

    return jsonify({"version": "default", "config": {}}), 200
```

## Versioning Config Bundles

Tag every update with a version number so clients know when to refresh:

```python
import time

def publish_config(platform: str, version: str, values: dict):
    key = f"config:{platform}:{version}"
    pipe = r.pipeline()
    for k, v in values.items():
        pipe.hset(key, k, v)
    pipe.hset(key, "_published_at", str(time.time()))
    pipe.execute()
```

## Client-Side Caching with ETag

Return an ETag so mobile clients can skip unnecessary downloads:

```python
import hashlib
import json

@app.get("/remote-config")
def get_remote_config():
    platform = request.args.get("platform", "ios")
    version = request.args.get("version", "v1")
    config = r.hgetall(f"config:{platform}:{version}") or {}

    etag = hashlib.md5(json.dumps(config, sort_keys=True).encode()).hexdigest()
    if request.headers.get("If-None-Match") == etag:
        return "", 304

    response = jsonify({"config": config})
    response.headers["ETag"] = etag
    return response
```

## Invalidating Config Cache

When you publish a new config, increment a global version counter so polling clients know to refresh:

```bash
INCR config:global:version
```

## Summary

Redis hashes store per-platform and per-version config bundles that mobile clients fetch via a lightweight REST endpoint. ETag-based caching prevents redundant data transfer. Publishing a config update takes milliseconds and is immediately available to all app instances - no server restart or deployment required.

