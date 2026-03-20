# How to Handle IPv6 in Rate Limiting Middleware

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Rate Limiting, Middleware, Security, Node.js, Nginx, Web Development

Description: Implement IPv6-aware rate limiting middleware that handles IPv6 address aggregation, prefix-based bucketing, and bypass prevention for web applications.

## Introduction

Rate limiting with IPv6 is more complex than IPv4 because a single attacker may own an entire /64 subnet (18 quintillion addresses) and rotate through them to bypass per-IP rate limits. Effective IPv6 rate limiting requires bucketing by network prefix rather than individual addresses.

## The IPv6 Rate Limiting Problem

With IPv4, an attacker has a limited pool of IPs. With IPv6:
- A single /48 allocation contains 65,536 /64 subnets
- Each /64 has 18,446,744,073,709,551,616 addresses
- Traditional per-IP rate limiting is easily bypassed

Solution: Rate limit by /48 or /64 prefix for IPv6, not individual addresses.

## Nginx: Rate Limiting by IPv6 Prefix

Nginx can map IPv6 addresses to a prefix using `geo` module:

```nginx
# /etc/nginx/conf.d/rate_limit.conf

# Map IPv6 addresses to their /64 prefix for rate limiting

geo $rate_limit_key {
    default $binary_remote_addr;  # For IPv4, use full address

    # For IPv6, extract /64 prefix (first 8 bytes)
    # This is handled differently - use a Lua map or upstream solution
}

# Simpler approach: use map to set key based on address family
map $remote_addr $rate_key {
    # IPv6 addresses match this pattern
    "~^([0-9a-fA-F:]+:){4}"  $1$2$3$4;  # First 4 groups (64-bit prefix)
    default                   $binary_remote_addr;
}

limit_req_zone $rate_key zone=api:10m rate=60r/m;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

## Node.js: express-rate-limit with IPv6 Prefix

```javascript
const rateLimit = require('express-rate-limit');
const net = require('net');

/**
 * Extract rate limiting key from IP address.
 * For IPv6, uses /64 prefix. For IPv4, uses full address.
 */
function getRateLimitKey(ip) {
  // Strip IPv4-mapped prefix
  const cleanIP = ip.replace(/^::ffff:/, '');

  if (!net.isIPv6(cleanIP)) {
    return cleanIP;  // IPv4: use full address
  }

  // IPv6: extract /64 prefix (first 4 groups of 4 hex chars)
  const groups = cleanIP.split(':');
  if (groups.length < 4) {
    // Expand compressed address
    const expanded = expandIPv6(cleanIP);
    return expanded.split(':').slice(0, 4).join(':') + '::/64';
  }

  return groups.slice(0, 4).join(':') + '::/64';
}

function expandIPv6(addr) {
  // Use Node's dns module to expand, or implement manually
  const parts = addr.split('::');
  if (parts.length === 2) {
    const left = parts[0].split(':').filter(x => x);
    const right = parts[1].split(':').filter(x => x);
    const missing = 8 - left.length - right.length;
    return [...left, ...Array(missing).fill('0'), ...right].join(':');
  }
  return addr;
}

// Configure rate limiter with IPv6-aware key
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,     // 1 minute window
  max: 100,                  // 100 requests per window per /64
  keyGenerator: (req) => {
    const ip = req.ip || req.socket.remoteAddress || '';
    const key = getRateLimitKey(ip);
    console.log(`Rate limit key for ${ip}: ${key}`);
    return key;
  },
  message: {
    error: 'Too many requests, please try again later.',
    retryAfter: '60 seconds'
  },
  standardHeaders: true,     // Send RateLimit-* headers
  legacyHeaders: false,
});

// Apply to all API routes
app.use('/api/', apiLimiter);
```

## Redis-Based IPv6 Rate Limiting

```python
import redis
import ipaddress
import time
from typing import Optional

class IPv6RateLimiter:
    def __init__(self, redis_client: redis.Redis, limit: int, window: int):
        self.redis = redis_client
        self.limit = limit        # Max requests
        self.window = window      # Window in seconds

    def get_rate_key(self, ip_str: str) -> str:
        """
        Return rate limiting key.
        IPv6: /64 prefix
        IPv4: full address
        """
        # Strip IPv4-mapped prefix
        ip_str = ip_str.replace('::ffff:', '')

        try:
            addr = ipaddress.ip_address(ip_str)
            if isinstance(addr, ipaddress.IPv6Address):
                # Get /64 network containing this address
                network = ipaddress.IPv6Network(f'{addr}/64', strict=False)
                return f'ratelimit:v6:{network.network_address}'
            else:
                return f'ratelimit:v4:{addr}'
        except ValueError:
            return f'ratelimit:unknown:{ip_str}'

    def is_allowed(self, ip: str) -> tuple[bool, int]:
        """
        Check if request is allowed.
        Returns (allowed, remaining_requests).
        """
        key = self.get_rate_key(ip)
        now = int(time.time())
        window_start = now - self.window

        # Sliding window using Redis sorted set
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, '-inf', window_start)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, self.window)
        results = pipe.execute()

        count = results[2]
        allowed = count <= self.limit
        remaining = max(0, self.limit - count)
        return allowed, remaining

# Usage in a web framework
limiter = IPv6RateLimiter(redis.Redis(), limit=100, window=60)

def rate_limit_middleware(ip: str) -> dict:
    allowed, remaining = limiter.is_allowed(ip)
    return {
        'allowed': allowed,
        'remaining': remaining,
        'key': limiter.get_rate_key(ip)
    }
```

## Testing IPv6 Rate Limiting

```bash
# Test that different /64 addresses share the same rate limit bucket
# All these should count toward the same /64 limit:
for i in 1 2 3 4 5; do
    curl -s -o /dev/null -w "%{http_code}\n" \
        --ipv6 -6 \
        http://[2001:db8::$i]/api/endpoint
done

# Test from a completely different /64 (should have its own bucket)
curl -6 http://[2001:db8:0:1::1]/api/endpoint
```

## Conclusion

IPv6 rate limiting must aggregate by network prefix (/48 or /64) rather than individual addresses to prevent bypass through address rotation. In Nginx, use `geo` or `map` directives to extract the prefix. In application code, normalize the client IP to its /64 prefix before using it as the rate limit key. Redis sorted sets provide efficient sliding-window rate limiting that scales with high request volumes.
