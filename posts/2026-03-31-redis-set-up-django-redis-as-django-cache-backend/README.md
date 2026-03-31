# How to Set Up django-redis as Django Cache Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Django, Caching, Django-Redis, Python

Description: Configure django-redis as the cache backend for your Django project to replace the default in-memory cache with a fast, persistent Redis store.

---

## Introduction

Django ships with a simple in-memory cache that is lost on server restart and is not shared across processes. `django-redis` replaces this with a Redis-backed cache that is persistent, shareable across application instances, and highly configurable. This guide walks you through installation, configuration, and common usage patterns.

## Installation

```bash
pip install django-redis
```

Make sure Redis is running and accessible from your Django application.

## Basic Configuration

Add the cache configuration to your `settings.py`:

```python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
```

Using database `1` instead of `0` keeps application cache separate from other Redis data.

## Using a Redis URL from Environment Variables

For production deployments, read the Redis URL from an environment variable:

```python
import os

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": os.environ.get("REDIS_URL", "redis://127.0.0.1:6379/1"),
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "PASSWORD": os.environ.get("REDIS_PASSWORD", ""),
        },
        "KEY_PREFIX": "myapp",
        "TIMEOUT": 300,  # Default TTL in seconds
    }
}
```

## Using the Cache in Views

```python
from django.core.cache import cache
from django.http import JsonResponse
from .models import Product

def product_list(request):
    cache_key = "product_list_all"
    products = cache.get(cache_key)

    if products is None:
        products = list(Product.objects.values("id", "name", "price"))
        cache.set(cache_key, products, timeout=600)

    return JsonResponse({"products": products})
```

## Cache Decorators

Use `cache_page` for whole-view caching:

```python
from django.views.decorators.cache import cache_page
from django.http import HttpResponse

@cache_page(60 * 15)  # Cache for 15 minutes
def expensive_view(request):
    # Expensive computation here
    result = compute_something()
    return HttpResponse(result)
```

## Low-Level Cache API

```python
from django.core.cache import cache

# Set with custom timeout
cache.set("user:42:profile", {"name": "Alice", "role": "admin"}, timeout=3600)

# Get with default fallback
profile = cache.get("user:42:profile", default={})

# Delete a key
cache.delete("user:42:profile")

# Get or set pattern
def get_user_stats(user_id):
    key = f"user:{user_id}:stats"
    stats = cache.get(key)
    if stats is None:
        stats = compute_user_stats(user_id)
        cache.set(key, stats, timeout=300)
    return stats
```

## Configuring Connection Pooling

```python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "CONNECTION_POOL_KWARGS": {
                "max_connections": 100,
                "retry_on_timeout": True,
            },
            "SOCKET_CONNECT_TIMEOUT": 5,
            "SOCKET_TIMEOUT": 5,
            "COMPRESSOR": "django_redis.compressors.zlib.ZlibCompressor",
        },
    }
}
```

## Accessing the Raw Redis Client

```python
from django_redis import get_redis_connection

def flush_user_cache(user_id):
    con = get_redis_connection("default")
    pattern = f"*:user:{user_id}:*"
    keys = con.keys(pattern)
    if keys:
        con.delete(*keys)
```

## Session Backend with Redis

Use the same Redis connection for sessions:

```python
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

## Summary

`django-redis` is a drop-in cache backend that replaces Django's in-memory cache with Redis, enabling cross-process cache sharing and persistence. Configuring it requires adding the `CACHES` setting in `settings.py` with the Redis URL and client options. Once installed, you can use all of Django's standard cache APIs including `cache_page`, `cache.set`, and `cache.get` without changing any application code.
