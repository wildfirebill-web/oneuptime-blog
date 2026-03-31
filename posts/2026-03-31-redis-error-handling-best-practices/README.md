# Redis Error Handling Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error Handling, Best Practice, Reliability, Production

Description: Learn how to handle Redis errors robustly - covering connection failures, command errors, timeouts, and graceful degradation strategies for production systems.

---

Redis errors in production range from transient network blips to persistent failures. Without proper error handling, a Redis outage can cascade into a full application failure. This guide covers practical patterns for handling Redis errors gracefully.

## Categorize Error Types

Redis errors fall into distinct categories that require different responses:

```python
from redis.exceptions import (
    ConnectionError,      # Network or server unreachable
    TimeoutError,         # Command or connect timeout
    ResponseError,        # Server returned an error (WRONGTYPE, etc.)
    AuthenticationError,  # Bad password
    BusyLoadingError,     # Server loading dataset
    ReadOnlyError         # Writing to a replica
)
```

## Implement Retry with Backoff for Transient Errors

Connection and timeout errors are often transient - retry them with exponential backoff:

```python
import time
import redis
from redis.exceptions import ConnectionError, TimeoutError

def redis_command_with_retry(client, command, *args, max_retries=3):
    for attempt in range(max_retries):
        try:
            return getattr(client, command)(*args)
        except (ConnectionError, TimeoutError) as e:
            if attempt == max_retries - 1:
                raise
            wait = 2 ** attempt * 0.1  # 0.1s, 0.2s, 0.4s
            time.sleep(wait)
```

## Graceful Degradation on Cache Miss

When Redis is unavailable, fall back to the source of truth rather than failing the request:

```python
def get_user(user_id: str):
    try:
        cached = redis_client.get(f"user:{user_id}")
        if cached:
            return json.loads(cached)
    except (ConnectionError, TimeoutError):
        # Log the error but continue
        logger.warning("Redis unavailable, fetching from DB")

    # Fall back to database
    return db.query("SELECT * FROM users WHERE id = ?", user_id)
```

## Handle WRONGTYPE Errors

A `WRONGTYPE` error occurs when you call a command on a key with a different data type. Catch and handle it specifically:

```python
from redis.exceptions import ResponseError

try:
    client.lpush("my_key", "value")
except ResponseError as e:
    if "WRONGTYPE" in str(e):
        # Key exists as a different type - delete and retry
        client.delete("my_key")
        client.lpush("my_key", "value")
    else:
        raise
```

## Use Circuit Breakers

Protect your application from repeated Redis failures using a circuit breaker pattern:

```python
class RedisCircuitBreaker:
    def __init__(self, threshold=5, timeout=30):
        self.failures = 0
        self.threshold = threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.open = False

    def call(self, func, *args, **kwargs):
        if self.open:
            if time.time() - self.last_failure_time > self.timeout:
                self.open = False  # Try half-open
            else:
                raise Exception("Circuit open - Redis unavailable")
        try:
            result = func(*args, **kwargs)
            self.failures = 0
            return result
        except (ConnectionError, TimeoutError):
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.threshold:
                self.open = True
            raise
```

## Log Errors with Context

Always log Redis errors with enough context to diagnose the issue:

```python
import logging

logger = logging.getLogger(__name__)

try:
    client.set("key", "value")
except Exception as e:
    logger.error(
        "Redis command failed",
        extra={
            "error_type": type(e).__name__,
            "command": "SET",
            "key": "key",
            "host": redis_host,
        }
    )
```

## Avoid Swallowing Errors Silently

The most dangerous pattern is silent error swallowing:

```python
# BAD - errors disappear
try:
    client.set("key", "value")
except Exception:
    pass

# GOOD - log and decide on fallback
try:
    client.set("key", "value")
except Exception as e:
    logger.warning(f"Redis write failed: {e}")
    # Decide: retry, fallback, or raise
```

## Summary

Categorize Redis errors by type and respond appropriately: retry transient errors with backoff, degrade gracefully when Redis is unavailable, and use circuit breakers to prevent cascading failures. Always log errors with context, and never silently swallow exceptions. Proper error handling ensures your application remains resilient even when Redis experiences issues.
