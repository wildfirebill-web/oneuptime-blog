# How to Configure Caddy as an IPv6 Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Caddy, Reverse Proxy, Automatic HTTPS, Dual-Stack

Description: Configure Caddy as an IPv6-capable reverse proxy with automatic HTTPS, dual-stack listeners, and proper IPv6 client IP handling in the Caddyfile and JSON config.

## Introduction

Caddy is a modern web server and reverse proxy that automatically obtains and renews TLS certificates. It listens on both IPv4 and IPv6 by default when the host has both address families configured. This guide covers dual-stack Caddyfile configuration, IPv6 backend proxying, and client IP handling.

## Basic Dual-Stack Caddyfile

```caddyfile
# /etc/caddy/Caddyfile

# Caddy automatically listens on both IPv4 and IPv6
app.example.com {
    # Automatic HTTPS with Let's Encrypt
    reverse_proxy backend:8080
}
```

By default, Caddy binds to `0.0.0.0` and `::` for all interfaces. No special IPv6 configuration is needed for dual-stack operation.

## Explicit IPv6 Binding

```caddyfile
# Bind only to a specific IPv6 address
{
    # Bind to specific IPv6 interface
    default_bind 2001:db8::1
}

app.example.com {
    bind 2001:db8::1
    reverse_proxy backend:8080
}

# Or bind to all IPv6 only
:443 {
    bind ::
    tls /etc/ssl/certs/cert.pem /etc/ssl/private/key.pem
    reverse_proxy backend:8080
}
```

## IPv6 Backend Proxying

```caddyfile
# Proxy to IPv6 backend — must use brackets
api.example.com {
    reverse_proxy {
        to [2001:db8::10]:8080
        to [2001:db8::11]:8080

        # Load balancing
        lb_policy round_robin

        # Health checks
        health_uri    /health
        health_interval 30s

        # Pass client IP headers
        header_up X-Forwarded-For {remote_host}
        header_up X-Real-IP {remote_host}
    }
}
```

## Multiple Backends (IPv4 + IPv6)

```caddyfile
# Mixed IPv4 and IPv6 upstream pool
backend.example.com {
    reverse_proxy {
        to 10.0.0.1:8080
        to 10.0.0.2:8080
        to [2001:db8::10]:8080
        to [2001:db8::11]:8080
    }
}
```

## Trusted Proxies for Real Client IP

```caddyfile
{
    # Global trusted proxy CIDRs
    servers {
        trusted_proxies static 10.0.0.0/8 fd00::/8 2001:db8:lb::/48
    }
}

app.example.com {
    # After trusted_proxies is set, {remote_host} contains real client IP
    reverse_proxy backend:8080 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
    }

    # Log with real IPv6 client address
    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

## IPv6 Rate Limiting

```caddyfile
# Rate limiting with Caddy (requires caddy-ratelimit plugin)
app.example.com {
    rate_limit {
        zone dynamic {remote_host} 100r/m
    }
    reverse_proxy backend:8080
}
```

## JSON API Configuration

For dynamic configuration via Caddy's API:

```json
{
  "apps": {
    "http": {
      "servers": {
        "main": {
          "listen": ["[::]:443", "0.0.0.0:443"],
          "routes": [{
            "match": [{"host": ["app.example.com"]}],
            "handle": [{
              "handler": "reverse_proxy",
              "upstreams": [
                {"dial": "[2001:db8::10]:8080"},
                {"dial": "[2001:db8::11]:8080"}
              ],
              "headers": {
                "request": {
                  "set": {
                    "X-Forwarded-For": ["{http.request.remote.host}"],
                    "X-Real-IP": ["{http.request.remote.host}"]
                  }
                }
              }
            }]
          }]
        }
      }
    }
  }
}
```

```bash
# Apply JSON config via Caddy API
curl -X POST "http://[::1]:2019/load" \
    -H "Content-Type: application/json" \
    -d @caddy-config.json
```

## Verify IPv6 Listening

```bash
# Check Caddy listens on IPv6
ss -tlnp | grep caddy
# Should show [::]:443 and [::]:80

# Test IPv6 connectivity
curl -6 https://app.example.com/health

# Check access logs for IPv6 addresses
tail -f /var/log/caddy/access.log | grep -E '"[0-9a-fA-F:]{3,39}"'
```

## Conclusion

Caddy handles IPv6 automatically — it listens on both `0.0.0.0` and `::` by default, and the `{remote_host}` placeholder captures IPv6 client addresses correctly without special configuration. For IPv6 backend proxying, use bracket notation for addresses in `to` directives. Configure `trusted_proxies` in the global block to enable correct client IP extraction from `X-Forwarded-For` when Caddy sits behind another proxy. Caddy is the easiest reverse proxy to make dual-stack since it requires no IPv6-specific configuration for basic operation.
