# How to Use Redis with Caddy Server for Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caddy, Caching

Description: Learn how to add Redis-backed HTTP response caching to Caddy Server using the cache-handler plugin to serve repeated requests from memory instead of hitting backends.

---

Caddy Server handles HTTPS automatically, but it does not cache responses out of the box. The `cache-handler` plugin adds HTTP caching support and can use Redis as the storage backend. This lets you cache API responses or static assets across Caddy instances.

## Install the Cache Plugin

Build Caddy with the `cache-handler` module using `xcaddy`:

```bash
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
xcaddy build \
  --with github.com/darkweak/souin/plugins/caddy
```

Or use the prebuilt Docker image with the plugin included:

```bash
docker pull darkweak/souin:latest-caddy
```

## Start Redis

```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

## Caddyfile Configuration

Configure Caddy to use the Souin cache middleware with Redis:

```text
{
    order cache before rewrite
    cache {
        ttl 5m
        stale 2m
        redis {
            url redis://localhost:6379
        }
    }
}

api.example.com {
    cache {
        ttl 300s
        key {
            headers X-User-ID
        }
    }
    reverse_proxy localhost:8080
}
```

The `key` block customizes the cache key. Including headers like `X-User-ID` lets you cache per-user responses.

## JSON Configuration (Alternative)

If you use Caddy's JSON config instead of Caddyfile:

```text
{
  "apps": {
    "http": {
      "servers": {
        "srv0": {
          "routes": [{
            "handle": [{
              "handler": "subroute",
              "routes": [{
                "handle": [{
                  "handler": "cache",
                  "ttl": "5m",
                  "storers": [{
                    "configuration": {
                      "url": "redis://localhost:6379"
                    },
                    "name": "redis"
                  }]
                }]
              }]
            }]
          }]
        }
      }
    }
  }
}
```

## Verifying Cached Responses

Make two requests and observe the cache header:

```bash
curl -I https://api.example.com/data
# X-Cache: MISS

curl -I https://api.example.com/data
# X-Cache: HIT
```

Inspect the cached entry in Redis:

```bash
redis-cli keys "souin*"
redis-cli get "souin_GET_api.example.com_/data"
```

## Cache Invalidation

Use Caddy's Souin API to invalidate specific cache entries:

```bash
curl -X PURGE https://api.example.com/data
```

Or expire keys directly in Redis:

```bash
redis-cli del "souin_GET_api.example.com_/data"
```

## Summary

Redis-backed caching in Caddy via the Souin/cache-handler plugin reduces backend load by storing HTTP responses in Redis. The Caddyfile configuration is straightforward, and the plugin supports per-header cache keys, TTL management, and PURGE endpoints. Combined with Caddy's automatic HTTPS, this gives you a capable reverse proxy with centralized caching in a compact setup.
