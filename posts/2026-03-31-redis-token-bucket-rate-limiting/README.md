# How to Implement Token Bucket Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, API

Description: Implement the token bucket algorithm for rate limiting with Redis to allow request bursting while enforcing average rate limits.

---

The token bucket algorithm is one of the most flexible rate limiting strategies. It allows short bursts of traffic while enforcing a sustained average rate. Each "bucket" fills with tokens at a steady rate and each request consumes one token. If the bucket is empty, the request is rejected.

## How Token Bucket Works

- Bucket capacity: maximum number of tokens (defines burst size)
- Refill rate: tokens added per second
- Each request removes one token
- Requests fail only when the bucket is empty

## Lua Script Implementation

Implementing token bucket in Redis requires an atomic operation. A Lua script ensures the read-modify-write cycle is atomic:

```lua
-- token_bucket.lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])          -- current timestamp (float)
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens based on elapsed time
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

if new_tokens >= requested then
    -- Allow request
    redis.call('HSET', key, 'tokens', new_tokens - requested, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
    return 1
else
    -- Deny request
    redis.call('HSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
    return 0
end
```

## Python Client

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

with open('token_bucket.lua', 'r') as f:
    BUCKET_SCRIPT = r.register_script(f.read())

def is_allowed(identifier: str, capacity: int = 10, refill_rate: float = 1.0) -> bool:
    key = f"bucket:{identifier}"
    now = time.time()
    result = BUCKET_SCRIPT(
        keys=[key],
        args=[capacity, refill_rate, now, 1]
    )
    return result == 1
```

## Usage in a Flask API

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/data')
def get_data():
    client_ip = request.remote_addr

    if not is_allowed(client_ip, capacity=20, refill_rate=5):
        return jsonify({"error": "Rate limit exceeded"}), 429

    return jsonify({"data": "your response here"})
```

## Testing the Bucket

```bash
# Check current bucket state
redis-cli HGETALL "bucket:192.168.1.1"

# Simulate burst of requests
for i in {1..25}; do
  python3 -c "from rate_limiter import is_allowed; print(is_allowed('test'))"
done
```

## Token Bucket vs Other Algorithms

```text
Algorithm       | Bursting | Precision | Complexity
----------------|----------|-----------|----------
Token Bucket    | Yes      | High      | Medium
Fixed Window    | Yes      | Low       | Low
Sliding Window  | Limited  | High      | Medium
Leaky Bucket    | No       | High      | Medium
```

## Summary

The token bucket algorithm is ideal when you want to allow short bursts while enforcing a sustainable average rate. Implementing it in Redis via a Lua script ensures the check-and-decrement operation is atomic, preventing race conditions under concurrent load. Set capacity equal to the allowed burst size and refill rate equal to your sustained requests-per-second target.
