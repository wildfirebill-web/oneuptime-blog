# How to Configure Dapr Binding with Redis

Author: [OneUptime](https://www.github.com/OneUptime)

Tags: Dapr, Binding, Redis, Cache, Output

Description: Configure the Dapr Redis output binding to execute Redis commands from microservices for cache operations, distributed counters, and list management without the Redis client library.

---

## Overview

The Dapr Redis binding is an output-only binding that supports a rich set of Redis commands including `get`, `set`, `delete`, `mget`, `mset`, `incr`, `expire`, and many more. This allows your services to interact with Redis as a cache, counter, or data structure store without embedding a Redis SDK.

```mermaid
flowchart LR
    App[Microservice] -->|POST /v1.0/bindings/redis| Sidecar[Dapr Sidecar]
    Sidecar -->|Redis Command| Redis[Redis Instance]
    Redis -->|Response| Sidecar
    Sidecar -->|Response| App
```

## Prerequisites

- Redis running locally or on Kubernetes
- Dapr CLI installed and initialized

## Deploy Redis

```bash
# Docker
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine

# Kubernetes with Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis \
  --namespace default \
  --set auth.enabled=true \
  --set auth.password=redispassword \
  --set replica.replicaCount=1
```

## Component Configuration

```yaml
# binding-redis.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: redis
  namespace: default
spec:
  type: bindings.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-master:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: redis-password
  - name: enableTLS
    value: "false"
  - name: failover
    value: "false"
```

```bash
kubectl create secret generic redis-secret \
  --from-literal=redis-password=redispassword \
  --namespace default

kubectl apply -f binding-redis.yaml
```

## Supported Operations

| Operation | Redis Command | Description |
|-----------|--------------|-------------|
| `get` | GET | Get a string value |
| `set` | SET | Set a string value |
| `delete` | DEL | Delete a key |
| `mget` | MGET | Get multiple keys |
| `mset` | MSET | Set multiple keys |
| `incr` | INCR | Increment a counter |
| `expire` | EXPIRE | Set key expiry |
| `hget` | HGET | Get hash field |
| `hset` | HSET | Set hash field |
| `llen` | LLEN | List length |
| `lpush` | LPUSH | Push to list head |

## SET and GET Operations

```bash
# Set a cache entry
curl -X POST http://localhost:3500/v1.0/bindings/redis \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "set",
    "data": "order-abc-status",
    "metadata": {
      "key": "order:abc:status",
      "ttlInSeconds": "3600"
    }
  }'

# Get the value
curl -X POST http://localhost:3500/v1.0/bindings/redis \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "get",
    "metadata": {
      "key": "order:abc:status"
    }
  }'
```

## Increment a Counter

```bash
# Increment a rate limit counter
curl -X POST http://localhost:3500/v1.0/bindings/redis \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "incr",
    "metadata": {
      "key": "ratelimit:user:alice"
    }
  }'
```

## MSET and MGET

```bash
# Set multiple keys at once
curl -X POST http://localhost:3500/v1.0/bindings/redis \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "mset",
    "data": [
      {"key": "config:timeout", "value": "30"},
      {"key": "config:retries", "value": "3"},
      {"key": "config:env", "value": "production"}
    ]
  }'

# Get multiple keys
curl -X POST http://localhost:3500/v1.0/bindings/redis \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "mget",
    "metadata": {
      "keys": "config:timeout,config:retries,config:env"
    }
  }'
```

## Python Application: Cache Service

```python
# cache_service.py
import json
import requests
from functools import wraps
from flask import Flask, request, jsonify

app = Flask(__name__)
DAPR_HTTP_PORT = 3500
BINDING_NAME = "redis"

def redis_exec(operation: str, data=None, metadata: dict = None):
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/bindings/{BINDING_NAME}"
    payload = {"operation": operation}
    if data is not None:
        payload["data"] = data
    if metadata:
        payload["metadata"] = metadata

    response = requests.post(url, json=payload)
    response.raise_for_status()
    return response.json() if response.text else None

def cache_set(key: str, value: str, ttl_seconds: int = 300):
    redis_exec("set", value, {"key": key, "ttlInSeconds": str(ttl_seconds)})

def cache_get(key: str) -> str | None:
    result = redis_exec("get", metadata={"key": key})
    return result.get("data") if result else None

def cache_delete(key: str):
    redis_exec("delete", metadata={"key": key})

def cache_incr(key: str) -> int:
    result = redis_exec("incr", metadata={"key": key})
    return int(result.get("data", 0)) if result else 0

@app.route('/product/<product_id>', methods=['GET'])
def get_product(product_id):
    cache_key = f"product:{product_id}"

    # Check cache first
    cached = cache_get(cache_key)
    if cached:
        return jsonify({"source": "cache", "data": json.loads(cached)})

    # Simulate database fetch
    product = {"id": product_id, "name": "Widget", "price": 29.99}

    # Cache for 5 minutes
    cache_set(cache_key, json.dumps(product), ttl_seconds=300)

    return jsonify({"source": "db", "data": product})

@app.route('/rate-check/<user_id>', methods=['POST'])
def rate_check(user_id):
    key = f"ratelimit:{user_id}"
    count = cache_incr(key)

    if count == 1:
        # Set TTL on first increment (1-minute window)
        redis_exec("expire", metadata={"key": key, "ttlInSeconds": "60"})

    if count > 10:
        return jsonify({"allowed": False, "count": count}), 429

    return jsonify({"allowed": True, "count": count})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

## Running Locally

```bash
dapr run \
  --app-id cache-service \
  --app-port 5001 \
  --dapr-http-port 3500 \
  --components-path ~/.dapr/components \
  -- python cache_service.py
```

## Summary

The Dapr Redis binding exposes Redis commands as output binding operations. Use `set`/`get`/`delete` for cache management, `incr` and `expire` for rate limiting, and `mset`/`mget` for bulk operations. Configure the binding with the Redis host and password, and send commands via a simple POST to `/v1.0/bindings/redis`. This approach keeps the Redis client library out of your application code and centralizes connection configuration.
