# How to Build a Rate Limiter in Python with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Rate Limiting, API Security, Sliding Window

Description: Learn how to build a Redis-backed rate limiter in Python using fixed window, sliding window, and token bucket algorithms with practical Flask integration.

---

## Why Redis for Rate Limiting?

Redis is the ideal backend for rate limiting because:
- Atomic increment operations prevent race conditions
- TTL-based key expiry handles window resets automatically
- In-memory speed minimizes latency overhead
- Works across multiple application instances (distributed)

## Fixed Window Counter

The simplest approach: count requests per time window using a TTL key.

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def is_rate_limited_fixed(user_id: str, limit: int = 100, window_seconds: int = 60) -> bool:
    """Returns True if the user has exceeded the rate limit."""
    key = f"rate:fixed:{user_id}:{int(time.time()) // window_seconds}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window_seconds)
    results = pipe.execute()

    count = results[0]
    return count > limit

# Usage
user_id = 'user:1001'
for i in range(5):
    limited = is_rate_limited_fixed(user_id, limit=3, window_seconds=60)
    print(f"Request {i+1}: {'BLOCKED' if limited else 'ALLOWED'}")
```

## Sliding Window Log

More accurate than fixed windows - tracks exact timestamps:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def is_rate_limited_sliding(user_id: str, limit: int = 100, window_seconds: int = 60) -> tuple:
    """Returns (is_limited, current_count)."""
    key = f"rate:sliding:{user_id}"
    now = time.time()
    window_start = now - window_seconds

    pipe = r.pipeline()
    # Remove entries outside the window
    pipe.zremrangebyscore(key, 0, window_start)
    # Count remaining entries
    pipe.zcard(key)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Set key expiry
    pipe.expire(key, window_seconds)
    results = pipe.execute()

    current_count = results[1] + 1  # +1 for current request
    is_limited = current_count > limit
    return is_limited, current_count

# Usage
for i in range(5):
    limited, count = is_rate_limited_sliding('user:1001', limit=3, window_seconds=10)
    status = 'BLOCKED' if limited else 'ALLOWED'
    print(f"Request {i+1}: {status} (count: {count})")
```

## Sliding Window Counter (Efficient)

Uses two counters for current and previous windows to approximate sliding window:

```python
import redis
import time
import math

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def is_rate_limited_hybrid(user_id: str, limit: int = 100, window_seconds: int = 60) -> bool:
    """Sliding window approximation using two fixed windows."""
    now = time.time()
    current_window = int(now) // window_seconds
    prev_window = current_window - 1

    current_key = f"rate:hybrid:{user_id}:{current_window}"
    prev_key = f"rate:hybrid:{user_id}:{prev_window}"

    pipe = r.pipeline()
    pipe.get(prev_key)
    pipe.incr(current_key)
    pipe.expire(current_key, window_seconds * 2)
    results = pipe.execute()

    prev_count = int(results[0] or 0)
    current_count = results[1]

    # Weight previous window by how far into the current window we are
    elapsed_in_window = now - (current_window * window_seconds)
    prev_weight = 1.0 - (elapsed_in_window / window_seconds)

    weighted_count = math.floor(prev_count * prev_weight) + current_count
    return weighted_count > limit
```

## Token Bucket Algorithm

Supports burst traffic while enforcing average rate:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def consume_token(user_id: str, capacity: int = 10, refill_rate: float = 1.0) -> bool:
    """
    Token bucket: capacity tokens max, refill_rate tokens/second.
    Returns True if request is allowed.
    """
    key = f"rate:tokens:{user_id}"
    now = time.time()

    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(key)
                bucket = pipe.hgetall(key)

                if bucket:
                    tokens = float(bucket.get('tokens', capacity))
                    last_refill = float(bucket.get('last_refill', now))
                else:
                    tokens = capacity
                    last_refill = now

                # Refill tokens based on elapsed time
                elapsed = now - last_refill
                tokens = min(capacity, tokens + elapsed * refill_rate)

                if tokens < 1:
                    pipe.unwatch()
                    return False  # Rate limited

                # Consume one token
                tokens -= 1

                pipe.multi()
                pipe.hset(key, mapping={
                    'tokens': tokens,
                    'last_refill': now
                })
                pipe.expire(key, int(capacity / refill_rate) + 1)
                pipe.execute()
                return True

            except redis.WatchError:
                continue

# Usage
for i in range(15):
    allowed = consume_token('user:1001', capacity=5, refill_rate=1.0)
    print(f"Request {i+1}: {'ALLOWED' if allowed else 'BLOCKED'}")
```

## Flask Middleware Integration

```python
from flask import Flask, request, jsonify
import redis
import time
import functools

app = Flask(__name__)
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def rate_limit(limit=60, window=60):
    """Rate limiting decorator for Flask routes."""
    def decorator(f):
        @functools.wraps(f)
        def decorated_function(*args, **kwargs):
            # Use IP or API key as identifier
            identifier = request.headers.get('X-API-Key') or request.remote_addr
            key = f"rate:{identifier}:{int(time.time()) // window}"

            pipe = r.pipeline()
            pipe.incr(key)
            pipe.expire(key, window)
            results = pipe.execute()
            count = results[0]

            if count > limit:
                return jsonify({
                    'error': 'Rate limit exceeded',
                    'limit': limit,
                    'window': window,
                    'retry_after': window - (int(time.time()) % window)
                }), 429

            # Add rate limit headers
            response = f(*args, **kwargs)
            if hasattr(response, 'headers'):
                response.headers['X-RateLimit-Limit'] = limit
                response.headers['X-RateLimit-Remaining'] = max(0, limit - count)
            return response
        return decorated_function
    return decorator

@app.route('/api/data')
@rate_limit(limit=10, window=60)
def get_data():
    return jsonify({'data': 'your response here'})
```

## Summary

Redis rate limiters in Python range from simple fixed window counters to sophisticated token bucket implementations. The fixed window approach with `INCR` and `EXPIRE` is easiest to implement, while the sliding window log using sorted sets gives the most accurate results. Token buckets are best when you need to accommodate short bursts. For production Flask or FastAPI applications, wrap rate limiting in a decorator that returns 429 responses with `Retry-After` headers.
