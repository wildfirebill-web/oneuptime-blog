# How to Configure Caddy Server for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Caddy, IPv6, Web Server, Reverse Proxy, Automatic HTTPS, Load Balancing

Description: A guide to configuring Caddy web server and reverse proxy with IPv6 support, including listening on IPv6 addresses and proxying to IPv6 backends.

Caddy automatically handles IPv6 in most configurations. By default, Caddy listens on `[::]` (all interfaces, including IPv6) unless configured otherwise. This guide covers explicitly configuring IPv6 and using IPv6 backends.

## Default Behavior

Caddy listens on both IPv4 and IPv6 by default:

```bash
# Caddy binds to :: (all interfaces) which accepts IPv4 and IPv6
# Verify with:
ss -6 -tlnp | grep caddy
# Expected: tcp6 *:80 and *:443
```

## Caddyfile: Listen on IPv6

```caddyfile
# Listen on all interfaces (default behavior)
example.com {
    reverse_proxy localhost:8080
}

# Explicitly listen on IPv6 only
[::]:80 {
    respond "IPv6 only server" 200
}

# Listen on specific IPv6 address
[2001:db8::proxy]:443 {
    tls internal
    reverse_proxy [2001:db8::backend]:8080
}

# Listen on multiple addresses including IPv6
http://0.0.0.0:8080, http://[::]:8080 {
    respond "Dual-stack server" 200
}
```

## Reverse Proxy to IPv6 Backends

```caddyfile
# Proxy to a single IPv6 backend
example.com {
    reverse_proxy [2001:db8::backend]:8080
    tls {
        on_demand
    }
}

# Load balance across IPv6 backends
api.example.com {
    reverse_proxy {
        to [2001:db8::server1]:8080
        to [2001:db8::server2]:8080
        to [2001:db8::server3]:8080

        lb_policy round_robin
        health_uri /health
        health_interval 10s
    }
}

# Mix IPv4 and IPv6 backends
mixed.example.com {
    reverse_proxy {
        to 10.0.0.1:8080
        to [2001:db8::server]:8080
        lb_policy least_conn
    }
}
```

## JSON Configuration for IPv6

```json
{
  "apps": {
    "http": {
      "servers": {
        "main": {
          "listen": [
            ":443",
            "[::]:443"
          ],
          "routes": [
            {
              "match": [
                {"host": ["example.com"]}
              ],
              "handle": [
                {
                  "handler": "reverse_proxy",
                  "upstreams": [
                    {"dial": "[2001:db8::backend1]:8080"},
                    {"dial": "[2001:db8::backend2]:8080"}
                  ],
                  "health_checks": {
                    "active": {
                      "uri": "/health",
                      "interval": "10s",
                      "timeout": "5s"
                    }
                  }
                }
              ]
            }
          ]
        }
      }
    }
  }
}
```

## Automatic HTTPS with IPv6

Caddy's automatic HTTPS works with IPv6:

```caddyfile
# Caddy automatically obtains TLS certificates
# DNS must have both A and AAAA records pointing to Caddy's IP
example.com {
    # Caddy resolves example.com, obtains cert, serves on both IPv4 and IPv6
    reverse_proxy [fd00:internal::backend]:3000
}
```

## IPv6 Real IP Headers

```caddyfile
api.example.com {
    reverse_proxy [2001:db8::backend]:8080 {
        # Preserve original IPv6 client IP
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

## Verify IPv6 Connectivity

```bash
# Test with curl over IPv6
curl -6 https://example.com/

# Check Caddy logs for IPv6 client connections
journalctl -u caddy -f | grep "::"

# Caddy access log shows IPv6 addresses
# {"level":"info","ts":"2026-03-20T...","msg":"handled request",
#   "remote_addr":"[2001:db8::client]:54321",...}
```

## IPv6-Only Caddy Server

```caddyfile
# For an IPv6-only deployment
{
    # Disable IPv4 listeners
    default_bind [::]
}

[::]:80 {
    redir https://{host}{uri}
}

[::]:443 {
    tls /etc/caddy/cert.pem /etc/caddy/key.pem
    reverse_proxy [2001:db8::backend]:8080
}
```

Caddy's zero-config approach to IPv6 — listening on both IPv4 and IPv6 by default and supporting IPv6 backend addresses with bracket notation — makes it one of the easiest web servers to use in dual-stack and IPv6-only environments.
