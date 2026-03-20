# How to Handle IPv6 Client Addresses in API Rate Limiting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Rate Limiting, API Gateway, Security, Networking, Nginx

Description: Implement effective API rate limiting for IPv6 clients by addressing prefix aggregation, address normalization, and per-/48 or per-/64 subnet bucketing strategies.

## Introduction

Rate limiting IPv6 clients presents a unique challenge: a single user can own an entire /48 or /64 prefix (millions of addresses) and trivially rotate through them to bypass per-IP limits. Effective IPv6 rate limiting requires bucketing by prefix, not individual address.

## The Problem: IPv6 Address Rotation

```text
# An attacker owns 2001:db8:1234::/48

# They can send from:
2001:db8:1234::1      -> rate limit bucket #1
2001:db8:1234::2      -> rate limit bucket #2 (bypass!)
2001:db8:1234:1::1    -> rate limit bucket #3 (bypass!)
```

The solution is to truncate the client address to its /64 or /48 and use that as the rate limit key.

## Strategy 1: NGINX Rate Limiting by /64

NGINX's `$binary_remote_addr` is per-IP. Create a map that extracts the /64 prefix.

```nginx
# nginx.conf - rate limit by IPv6 /64 prefix

# Extract the first 8 bytes (64 bits) of the IPv6 address
# For IPv4 clients, the full address is used
geo $limit_key {
    default $binary_remote_addr;
    # For IPv6, use a /64 prefix mask
}

# A more portable approach using a Lua/njs snippet or map:
map $remote_addr $rate_limit_key {
    # Strip last 64 bits from IPv6 (simplistic heuristic via regex)
    "~^([0-9a-f:]+:[0-9a-f:]+:[0-9a-f:]+:[0-9a-f:]+):" "$1::/64";
    default $remote_addr;
}

limit_req_zone $rate_limit_key zone=api_zone:10m rate=100r/m;

server {
    listen 80;
    listen [::]:80;

    location /api/ {
        limit_req zone=api_zone burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

## Strategy 2: Python - Normalize IPv6 to /64 for Rate Limiting

```python
import ipaddress
from functools import lru_cache

@lru_cache(maxsize=10000)
def get_rate_limit_key(client_ip: str, prefix_len: int = 64) -> str:
    """
    Normalize an IP address to a subnet key for rate limiting.
    IPv6 addresses are truncated to the given prefix length.
    IPv4 addresses are returned as-is.
    """
    try:
        addr = ipaddress.ip_address(client_ip)
        if isinstance(addr, ipaddress.IPv6Address):
            # Create a network from the address, masked to prefix_len
            network = ipaddress.IPv6Network(
                f"{client_ip}/{prefix_len}", strict=False
            )
            return str(network.network_address)
        else:
            # IPv4: rate limit per individual address
            return client_ip
    except ValueError:
        # Invalid IP - use as-is
        return client_ip


# Example usage in a Flask view
from flask import request
import redis

r = redis.Redis()

def check_rate_limit(max_requests=100, window=60):
    client_ip = request.remote_addr
    key = f"ratelimit:{get_rate_limit_key(client_ip)}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()

    return count <= max_requests
```

## Strategy 3: Redis-Based Rate Limiting with /48 Bucketing

For higher-value APIs, bucket at /48 to handle more sophisticated address rotation.

```python
def get_ipv6_prefix(ip: str, prefix_len: int = 48) -> str:
    """Return the /48 network prefix for an IPv6 address."""
    try:
        net = ipaddress.IPv6Network(f"{ip}/{prefix_len}", strict=False)
        return str(net.network_address)
    except ValueError:
        return ip

# Redis Lua script for atomic rate limiting
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end
if current > limit then
    return 0
end
return 1
"""

def is_allowed(client_ip, limit=1000, window=3600):
    prefix = get_ipv6_prefix(client_ip, prefix_len=48)
    key = f"rl:{prefix}"
    result = r.eval(RATE_LIMIT_SCRIPT, 1, key, limit, window)
    return bool(result)
```

## Strategy 4: Kong Plugin Configuration

```yaml
# Kong rate-limiting-advanced plugin with IPv6 bucketing
plugins:
  - name: rate-limiting-advanced
    config:
      limit: [100]
      window_size: [60]
      # Use consumer IP, supports IPv6 natively
      identifier: ip
      # Kong normalizes IPv6 internally
      sync_rate: 10
```

## Conclusion

IPv6 rate limiting must operate at the subnet level, not individual address level. /64 bucketing is the minimum effective prefix; /48 provides stronger protection. Whichever gateway or framework you use, normalize IPv6 to a consistent prefix before creating rate limit keys. Use OneUptime to monitor your APIs for anomalous traffic patterns across both IP families.
