# How to Handle Redis Connection Failures in Flask

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Flask, Error Handling, Resilience, Fallback

Description: Learn how to gracefully handle Redis connection failures in Flask with fallback strategies, circuit breakers, and health checks for resilient applications.

---

Flask applications that depend on Redis for caching, sessions, or rate limiting need a plan for when Redis becomes unavailable. Graceful degradation keeps your app serving requests instead of returning 500 errors.

## Catching Connection Exceptions

The `redis` package raises `redis.exceptions.ConnectionError` and `redis.exceptions.TimeoutError` on failure:

```python
import redis
from flask import Flask, jsonify

app = Flask(__name__)
r = redis.Redis(host="localhost", port=6379, socket_connect_timeout=2, socket_timeout=2)

@app.route("/api/data")
def get_data():
    try:
        cached = r.get("data:all")
        if cached:
            return jsonify({"source": "cache", "data": cached.decode()})
    except (redis.exceptions.ConnectionError, redis.exceptions.TimeoutError) as e:
        app.logger.warning("Redis unavailable: %s", e)

    # Fall back to database
    data = fetch_from_db()
    return jsonify({"source": "db", "data": data})
```

## Flask-Caching Fallback

Flask-Caching can fall back to a null cache when Redis is down:

```python
from flask_caching import Cache

PRIMARY_CACHE = {
    "CACHE_TYPE": "RedisCache",
    "CACHE_REDIS_URL": "redis://localhost:6379/0",
    "CACHE_DEFAULT_TIMEOUT": 300,
}

FALLBACK_CACHE = {
    "CACHE_TYPE": "NullCache",  # no-op: all reads return None, writes ignored
}

cache = Cache()

def create_app():
    app = Flask(__name__)
    try:
        r = redis.Redis(host="localhost", port=6379, socket_connect_timeout=1)
        r.ping()
        app.config.update(PRIMARY_CACHE)
        app.logger.info("Using Redis cache")
    except Exception:
        app.config.update(FALLBACK_CACHE)
        app.logger.warning("Redis unavailable, using NullCache fallback")
    cache.init_app(app)
    return app
```

## Simple Circuit Breaker

Avoid flooding a down Redis with repeated connection attempts:

```python
import time
from threading import Lock

class SimpleCircuitBreaker:
    CLOSED, OPEN = "closed", "open"

    def __init__(self, failure_threshold=5, reset_timeout=30):
        self.state = self.CLOSED
        self.failure_count = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.last_failure_time = None
        self._lock = Lock()

    def call(self, fn, fallback=None):
        with self._lock:
            if self.state == self.OPEN:
                if time.time() - self.last_failure_time > self.reset_timeout:
                    self.state = self.CLOSED
                    self.failure_count = 0
                else:
                    return fallback() if fallback else None

        try:
            result = fn()
            with self._lock:
                self.failure_count = 0
            return result
        except Exception as e:
            with self._lock:
                self.failure_count += 1
                self.last_failure_time = time.time()
                if self.failure_count >= self.threshold:
                    self.state = self.OPEN
            if fallback:
                return fallback()
            raise

redis_breaker = SimpleCircuitBreaker()

def get_cached_user(user_id):
    key = f"user:{user_id}"
    return redis_breaker.call(
        fn=lambda: r.get(key),
        fallback=lambda: None
    )
```

## Health Check Endpoint

```python
@app.route("/health")
def health():
    status = {"app": "ok", "redis": "ok"}
    code = 200

    try:
        r.ping()
    except Exception as e:
        status["redis"] = f"error: {e}"
        code = 503

    return jsonify(status), code
```

## Connection Pool Configuration

```python
pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    max_connections=20,
    socket_connect_timeout=2,
    socket_timeout=2,
    retry_on_timeout=True,
)
r = redis.Redis(connection_pool=pool)
```

Setting `retry_on_timeout=True` retries once on timeout automatically.

## Summary

Handle Redis failures in Flask by wrapping Redis calls in try/except for `ConnectionError` and `TimeoutError`, providing meaningful fallbacks (database reads, default values). Use Flask-Caching's `NullCache` as a startup fallback, implement a circuit breaker to avoid retry storms, and expose a `/health` endpoint so load balancers can route around unhealthy instances. Configure short `socket_connect_timeout` and `socket_timeout` so failures are detected quickly.
