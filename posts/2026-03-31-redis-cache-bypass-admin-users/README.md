# How to Implement Cache Bypass for Admin Users in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Authentication

Description: Learn how to bypass Redis cache for admin users so they always see fresh data while regular users benefit from cached responses.

---

Serving cached data to admin users can cause confusion - admins need real-time accuracy for moderation, auditing, and data management. Implementing a cache bypass for admin users lets you keep caching benefits for regular traffic while ensuring admins always get live data.

## How Cache Bypass Works

The approach is simple: inspect the incoming request to determine if the user has admin privileges. If they do, skip the cache read and go directly to the source of truth. Optionally, you can also skip writing the response back to cache.

## Implementation

Here is a Python/Flask example using Redis:

```python
import redis
import json
from functools import wraps
from flask import request, g

r = redis.Redis(host='localhost', port=6379, db=0)

def cached(ttl=300, bypass_for_admin=True):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            is_admin = getattr(g, 'user_is_admin', False)

            # Bypass cache for admin users
            if bypass_for_admin and is_admin:
                return func(*args, **kwargs)

            cache_key = f"cache:{request.path}:{request.query_string.decode()}"
            cached_value = r.get(cache_key)

            if cached_value:
                return json.loads(cached_value)

            result = func(*args, **kwargs)
            r.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator
```

Apply the decorator to your route handlers:

```python
@app.route('/api/products')
@cached(ttl=600)
def get_products():
    return fetch_products_from_db()
```

## Middleware Approach

For larger applications, handle bypass logic in middleware:

```python
class CacheBypassMiddleware:
    def __init__(self, app, redis_client):
        self.app = app
        self.redis = redis_client

    def __call__(self, environ, start_response):
        user_role = environ.get('HTTP_X_USER_ROLE', '')
        if user_role == 'admin':
            environ['CACHE_BYPASS'] = True
        return self.app(environ, start_response)
```

## Checking Admin Status via JWT

If your app uses JWT tokens, extract the role claim:

```python
import jwt

def get_user_from_token(token: str) -> dict:
    payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    return payload

def should_bypass_cache(request) -> bool:
    auth_header = request.headers.get('Authorization', '')
    if not auth_header.startswith('Bearer '):
        return False
    token = auth_header[7:]
    user = get_user_from_token(token)
    return user.get('role') == 'admin'
```

## Cache Key Tagging

You can also tag cache entries by user role, serving different cached responses per role tier:

```bash
# Regular user cache key
SET "cache:products:user" "[...]" EX 600

# Admin-specific cache (shorter TTL or not cached at all)
# Simply omit writing this key for admin requests
```

## Verifying Behavior

Test your bypass logic in the Redis CLI:

```bash
# Confirm regular user hits cache
redis-cli GET "cache:/api/products:"

# After admin request, confirm no new key was written for admin
redis-cli KEYS "cache:*"
```

## Summary

Cache bypass for admin users is a lightweight strategy that trades a tiny performance overhead for data accuracy. By checking admin status before every cache read, admins always get live results. Regular users continue to benefit from low-latency cached responses, keeping your overall system fast and reliable.
