# How to Build a Rate Limiter in Ruby with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ruby, Rate Limiting, Lua, Rack

Description: Learn how to build a fixed window and sliding window rate limiter in Ruby using Redis atomic operations and Lua scripts, with Rack middleware integration.

---

Redis is the ideal backend for distributed rate limiting in Ruby because its atomic operations work correctly across multiple application processes. This guide covers a fixed window counter and a Lua-based sliding window implementation.

## Fixed Window Rate Limiter

```ruby
require 'redis'

class FixedWindowRateLimiter
  def initialize(redis)
    @redis = redis
  end

  def allowed?(client_id, limit:, window_seconds:)
    window = Time.now.to_i / window_seconds
    key    = "rate:#{client_id}:#{window}"

    count = @redis.incr(key)
    @redis.expire(key, window_seconds) if count == 1

    count <= limit
  end
end
```

Usage:

```ruby
redis   = Redis.new
limiter = FixedWindowRateLimiter.new(redis)

if limiter.allowed?(request.ip, limit: 100, window_seconds: 60)
  # process request
else
  # return 429
end
```

## Sliding Window Rate Limiter with Lua

Lua scripts run atomically in Redis, preventing race conditions between the ZADD and ZCARD operations:

```ruby
SLIDING_WINDOW_SCRIPT = <<~LUA
  local key    = KEYS[1]
  local now    = tonumber(ARGV[1])
  local window = tonumber(ARGV[2]) * 1000
  local limit  = tonumber(ARGV[3])
  local uid    = ARGV[4]

  redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
  local count = redis.call('ZCARD', key)

  if count < limit then
    redis.call('ZADD', key, now, uid)
    redis.call('PEXPIRE', key, window)
    return 1
  end
  return 0
LUA

class SlidingWindowRateLimiter
  def initialize(redis)
    @redis  = redis
    @sha    = redis.script(:load, SLIDING_WINDOW_SCRIPT)
  end

  def allowed?(client_id, limit:, window_seconds:)
    key    = "sliding:#{client_id}"
    now_ms = (Time.now.to_f * 1000).to_i
    uid    = SecureRandom.hex(8)

    result = @redis.evalsha(@sha, keys: [key], argv: [now_ms, window_seconds, limit, uid])
    result == 1
  end
end
```

## Rack Middleware

```ruby
class RateLimitMiddleware
  def initialize(app, redis:, limit: 100, window: 60)
    @app     = app
    @limiter = SlidingWindowRateLimiter.new(redis)
    @limit   = limit
    @window  = window
  end

  def call(env)
    request = Rack::Request.new(env)
    client_id = request.ip

    if @limiter.allowed?(client_id, limit: @limit, window_seconds: @window)
      @app.call(env)
    else
      [
        429,
        { 'Content-Type' => 'application/json', 'Retry-After' => @window.to_s },
        [JSON.generate({ error: 'Too Many Requests', retry_after: @window })]
      ]
    end
  end
end
```

Mounting in a Rack app:

```ruby
# config.ru
require 'redis'
require_relative 'rate_limit_middleware'

redis = Redis.new

use RateLimitMiddleware, redis: redis, limit: 60, window: 60
run MyApp
```

## Rails Integration

```ruby
# config/initializers/rate_limiter.rb
$rate_limiter = SlidingWindowRateLimiter.new(Redis.new)

# app/controllers/application_controller.rb
before_action :check_rate_limit

def check_rate_limit
  ip = request.remote_ip
  return if $rate_limiter.allowed?(ip, limit: 100, window_seconds: 60)

  render json: { error: 'Too Many Requests' }, status: :too_many_requests
end
```

## Per-Endpoint Limits

```ruby
# Stricter limit for auth endpoints
limit = request.path.start_with?('/api/login') ? 5 : 100
limiter.allowed?(request.ip, limit: limit, window_seconds: 60)
```

## Summary

Redis rate limiters in Ruby are clean to implement with a fixed window INCR counter for simplicity or a Lua sliding window for precision. Rack middleware makes it easy to apply rate limits across all routes. Both approaches work correctly across multiple Puma workers and servers because all state is stored atomically in Redis.
