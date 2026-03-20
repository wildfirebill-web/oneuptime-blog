# How to Rate Limit IPv6 Clients at the Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Rate Limiting, Nginx, HAProxy, Reverse Proxy, Security

Description: Configure rate limiting for IPv6 clients at the reverse proxy layer, accounting for the challenge that IPv6 clients may share a /64 prefix, requiring prefix-based rather than per-address limiting.

## Introduction

Rate limiting IPv6 clients is more complex than IPv4 because a single user can legitimately rotate between thousands of addresses within a /64 prefix. Rate limiting by individual /128 address is ineffective — an attacker cycles through addresses while a legitimate mobile user gets blocked. The correct approach is to rate limit by /48 or /64 prefix at the reverse proxy.

## The IPv6 Rate Limiting Challenge

```
IPv4: 1 user = 1 IP (e.g., 203.0.113.5)
IPv6: 1 user = /64 prefix with 2^64 addresses
      e.g., 2001:db8:cafe:1::/64

Naive per-/128 limit: ineffective against prefix rotation
Correct approach: rate limit by /48 or /64 prefix
```

## Nginx: Rate Limit by IPv6 /48 Prefix

```nginx
# /etc/nginx/nginx.conf

http {
    # Custom variable: extract /48 prefix from IPv6 address
    # For IPv4 addresses, use the full address
    geo $limit_key {
        default $binary_remote_addr;
    }

    # Map to extract /48 prefix for IPv6
    map $remote_addr $ipv6_prefix {
        # If it's an IPv6 address, mask to /48
        ~^([0-9a-f]{1,4}:[0-9a-f]{1,4}:[0-9a-f]{1,4}):  "$1::/48";
        default $remote_addr;
    }

    # Rate limit zone keyed on IPv6 /48 prefix (or full IPv4 address)
    limit_req_zone $ipv6_prefix zone=per_prefix:10m rate=60r/m;

    server {
        listen [::]:443 ssl;
        listen 443 ssl;

        location /api/ {
            # Apply rate limit: burst allows short spikes
            limit_req zone=per_prefix burst=20 nodelay;
            limit_req_status 429;

            proxy_pass http://backend:8080;
            proxy_set_header X-Forwarded-For $remote_addr;
        }
    }
}
```

## Nginx: Lua-Based IPv6 Prefix Rate Limiting (OpenResty)

```nginx
# OpenResty / Nginx with lua-resty-limit-traffic

lua_shared_dict limit_store 10m;

server {
    listen [::]:443 ssl;

    location /api/ {
        access_by_lua_block {
            local limit_req = require "resty.limit.req"
            local ipaddr = ngx.var.remote_addr

            -- Extract /48 prefix for IPv6
            local prefix = ipaddr
            local count = 0
            for _ in string.gmatch(ipaddr, ":") do
                count = count + 1
            end

            if count >= 2 then
                -- IPv6: truncate to /48 (first 3 groups)
                local groups = {}
                for g in string.gmatch(ipaddr, "([^:]+)") do
                    table.insert(groups, g)
                    if #groups == 3 then break end
                end
                prefix = table.concat(groups, ":") .. "::/48"
            end

            local lim, err = limit_req.new("limit_store", 60, 20)
            if not lim then
                ngx.log(ngx.ERR, "failed to create limiter: ", err)
                return
            end

            local delay, err = lim:incoming(prefix, true)
            if not delay then
                if err == "rejected" then
                    ngx.exit(429)
                end
            end
        }

        proxy_pass http://backend:8080;
    }
}
```

## HAProxy: Rate Limit by IPv6 /48 Prefix

```haproxy
# /etc/haproxy/haproxy.cfg

frontend web_ipv6
    bind [::]:443 ssl crt /etc/ssl/certs/app.pem

    # Extract IPv6 /48 prefix using ACL
    # Use src_get_gpc0 for tracking
    stick-table type ipv6 size 1m expire 60s store conn_rate(60s),http_req_rate(60s)

    # Track by /48 prefix — HAProxy uses full address but can mask
    # For /48 masking: use custom maps or track per connection
    tcp-request connection track-sc0 src mask ffff:ffff:ffff::

    # Rate limit: allow max 100 HTTP requests per 60 seconds per /48
    http-request track-sc1 src mask ffff:ffff:ffff::
    http-request deny deny_status 429 if { sc_http_req_rate(1) gt 100 }

    default_backend app

backend app
    server app1 [2001:db8::10]:8080
```

## Traefik: Rate Limit Middleware for IPv6

```yaml
# traefik-rate-limit.yml

http:
  middlewares:
    ipv6-rate-limit:
      rateLimit:
        # Traefik rate limits per source IP by default
        # For IPv6 prefix grouping, use a custom plugin or external solution
        average: 60
        burst: 20
        period: 1m
        sourceCriterion:
          ipStrategy:
            # Trust the load balancer in front
            depth: 1

  routers:
    api:
      rule: "PathPrefix(`/api`)"
      middlewares:
        - ipv6-rate-limit
      service: backend
```

## Application-Level IPv6 Prefix Rate Limiting

```python
#!/usr/bin/env python3
# ipv6_rate_limiter.py

import ipaddress
import time
from collections import defaultdict

class IPv6PrefixRateLimiter:
    """Rate limiter that groups IPv6 addresses by /48 prefix."""

    def __init__(self, max_requests: int = 100, window_seconds: int = 60,
                 ipv6_prefix_length: int = 48):
        self.max_requests = max_requests
        self.window = window_seconds
        self.prefix_length = ipv6_prefix_length
        self._counters: dict = defaultdict(list)

    def _get_key(self, ip_str: str) -> str:
        """Map an IP address to a rate limiting key."""
        try:
            ip = ipaddress.ip_address(ip_str)
            if isinstance(ip, ipaddress.IPv6Address):
                # Group by /48 prefix
                network = ipaddress.ip_network(
                    f"{ip_str}/{self.prefix_length}", strict=False
                )
                return str(network.network_address)
        except ValueError:
            pass
        return ip_str  # Fall back to exact IP

    def is_allowed(self, ip_str: str) -> bool:
        key = self._get_key(ip_str)
        now = time.time()

        # Remove expired timestamps
        self._counters[key] = [
            t for t in self._counters[key] if now - t < self.window
        ]

        if len(self._counters[key]) >= self.max_requests:
            return False

        self._counters[key].append(now)
        return True

# Usage
limiter = IPv6PrefixRateLimiter(max_requests=100, window_seconds=60)

def check_rate_limit(client_ip: str) -> bool:
    if not limiter.is_allowed(client_ip):
        print(f"Rate limited: {client_ip}")
        return False
    return True
```

## Conclusion

IPv6 rate limiting at the reverse proxy must operate on network prefixes (/48 or /64) rather than individual addresses to be effective. Nginx supports this via `limit_req_zone` with a mapped prefix variable or Lua scripting in OpenResty. HAProxy can mask source addresses with `mask ffff:ffff:ffff::` in stick tables. The correct prefix length depends on your threat model: /64 for consumer ISPs (SLAAC), /48 for enterprise allocations. Always rate limit from the trusted `$remote_addr` rather than X-Forwarded-For to prevent spoofing.
