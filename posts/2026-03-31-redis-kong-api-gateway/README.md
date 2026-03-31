# How to Use Redis with Kong API Gateway

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kong, API Gateway, Rate Limiting, Cache

Description: Learn how to configure Kong API Gateway to use Redis as a shared backend for rate limiting and response caching across multiple Kong nodes.

---

Kong uses Redis as a shared data store for plugins that require cross-node consistency: rate limiting counters and proxy cache storage. Without Redis, each Kong node maintains its own counters and cache, leading to inaccurate rate limits in a multi-node deployment.

## Why Redis with Kong?

```text
Without Redis:
  - Kong node 1 allows 100 req/min
  - Kong node 2 also allows 100 req/min
  - Client gets 200 req/min (double the limit)

With Redis:
  - All Kong nodes share rate limit counters
  - Client gets exactly 100 req/min regardless of which node is hit
```

## Setting Up Kong with Redis via Docker Compose

```yaml
version: "3.8"
services:
  redis:
    image: redis:7
    ports:
      - "6379:6379"

  kong:
    image: kong:3.5
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong.yml
    ports:
      - "8000:8000"
      - "8001:8001"
    volumes:
      - ./kong.yml:/kong.yml
```

## Kong Declarative Config with Rate Limiting + Redis

```yaml
# kong.yml
_format_version: "3.0"

services:
  - name: my-api
    url: http://api-backend:3000
    routes:
      - name: api-route
        paths:
          - /api

plugins:
  - name: rate-limiting
    service: my-api
    config:
      minute: 100
      hour: 2000
      policy: redis
      redis_host: redis
      redis_port: 6379
      redis_timeout: 2000
      redis_database: 0

  - name: proxy-cache
    service: my-api
    config:
      response_code: [200]
      request_method: [GET, HEAD]
      content_type: ["application/json"]
      cache_ttl: 300
      storage_ttl: 600
      strategy: redis
      redis:
        host: redis
        port: 6379
        timeout: 2000
        database: 1
```

## Verifying Redis Rate Limit Keys

```bash
# After making requests, check Kong's rate limit keys in Redis
redis-cli KEYS "ratelimit:*"
redis-cli KEYS "*rate*"

# Typical key format: {ip}:{minute}
redis-cli SCAN 0 MATCH "*rate*" COUNT 20
```

## Key Namespace Separation

Kong rate limiting uses database 0 and proxy-cache uses database 1 in the config above. You can also use a dedicated Redis instance per plugin.

```yaml
# Separate Redis for rate limiting
plugins:
  - name: rate-limiting
    config:
      policy: redis
      redis_host: redis-ratelimit.internal
      redis_port: 6379

  - name: proxy-cache
    config:
      strategy: redis
      redis:
        host: redis-cache.internal
        port: 6379
```

## Monitoring Rate Limit Keys

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_rate_limit_usage(ip_address: str) -> dict:
    import time
    minute_key = f"ratelimit:{ip_address}:minute:{int(time.time() // 60)}"
    count = r.get(minute_key)
    return {
        "ip": ip_address,
        "requests_this_minute": int(count) if count else 0,
        "limit": 100
    }
```

## Testing the Configuration

```bash
# Start the stack
docker-compose up -d

# Test rate limiting
for i in $(seq 1 110); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/api/test
done
# First 100: 200 OK, next 10: 429 Too Many Requests

# Check cached response headers
curl -v http://localhost:8000/api/products | grep X-Cache
```

## Summary

Kong uses Redis as a shared store for rate limiting counters and proxy cache data across all gateway nodes, ensuring consistent enforcement in multi-node deployments. Configure the `rate-limiting` plugin with `policy: redis` and the `proxy-cache` plugin with `strategy: redis`, pointing both at the same or separate Redis instances. Separate Redis databases or instances per plugin prevents key collisions and allows independent eviction policies.

