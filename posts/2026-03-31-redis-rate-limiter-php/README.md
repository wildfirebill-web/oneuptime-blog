# How to Build a Rate Limiter in PHP with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PHP, Rate Limiting, Lua, API

Description: Learn how to build a fixed window and sliding window rate limiter in PHP with Redis using atomic Lua scripts for accurate per-client request limiting.

---

Redis is an ideal rate limiting backend for PHP applications because its atomic operations work reliably across multiple server instances. This guide covers both a fixed window counter and a sliding window approach.

## Fixed Window Rate Limiter

```php
class FixedWindowRateLimiter
{
    private Redis $redis;

    public function __construct(Redis $redis)
    {
        $this->redis = $redis;
    }

    public function isAllowed(string $clientId, int $limit, int $windowSeconds): bool
    {
        $window = (int) floor(time() / $windowSeconds);
        $key    = "rate:{$clientId}:{$window}";

        $count = $this->redis->incr($key);

        if ($count === 1) {
            $this->redis->expire($key, $windowSeconds);
        }

        return $count <= $limit;
    }
}
```

Usage:

```php
$redis   = new Redis();
$redis->connect('127.0.0.1', 6379);
$limiter = new FixedWindowRateLimiter($redis);

$ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';

if (!$limiter->isAllowed($ip, limit: 100, windowSeconds: 60)) {
    http_response_code(429);
    header('Retry-After: 60');
    echo json_encode(['error' => 'Too Many Requests']);
    exit;
}
```

## Sliding Window Rate Limiter with Lua

Lua scripts run atomically and provide more accurate burst limiting:

```php
class SlidingWindowRateLimiter
{
    private Redis $redis;

    private string $luaScript = <<<'LUA'
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
LUA;

    public function __construct(Redis $redis)
    {
        $this->redis = $redis;
    }

    public function isAllowed(string $clientId, int $limit, int $windowSeconds): bool
    {
        $key    = "sliding:{$clientId}";
        $nowMs  = (int) (microtime(true) * 1000);
        $uid    = uniqid('', true);

        $result = $this->redis->eval(
            $this->luaScript,
            [$key, $nowMs, $windowSeconds, $limit, $uid],
            1
        );

        return $result === 1;
    }
}
```

## Middleware Example (Plain PHP)

```php
function rateLimitMiddleware(SlidingWindowRateLimiter $limiter): void
{
    $ip = $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    $route = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    $key = "{$ip}:{$route}";

    if (!$limiter->isAllowed($key, limit: 30, windowSeconds: 60)) {
        header('HTTP/1.1 429 Too Many Requests');
        header('Content-Type: application/json');
        header('Retry-After: 60');
        echo json_encode(['error' => 'Too Many Requests', 'retry_after' => 60]);
        exit;
    }
}
```

## Per-Route or Per-User Limits

```php
// More strict limit for login endpoint
$isLoginRoute = ($_SERVER['REQUEST_URI'] ?? '') === '/api/login';
$limit = $isLoginRoute ? 5 : 100;

$limiter->isAllowed($ip, $limit, windowSeconds: 60);
```

## Getting the Remaining Count

```php
public function remaining(string $clientId, int $limit, int $windowSeconds): int
{
    $window = (int) floor(time() / $windowSeconds);
    $count  = (int) $this->redis->get("rate:{$clientId}:{$window}");
    return max(0, $limit - $count);
}
```

## Summary

Redis-backed rate limiters in PHP are straightforward to implement. The fixed window counter suits most use cases, while the Lua-based sliding window provides more accurate protection against bursty traffic. Both approaches work across multiple PHP processes and servers because Redis handles all state atomically.
