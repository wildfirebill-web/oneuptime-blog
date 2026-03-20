# How to Configure Caddy HTTP/3 with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Caddy, HTTP/3, QUIC, IPv6, Web Server, TLS, Automatic HTTPS

Description: Configure Caddy to serve HTTP/3 over QUIC with IPv6 support, taking advantage of Caddy's automatic HTTPS and native HTTP/3 implementation for low-latency connections.

---

Caddy enables HTTP/3 by default when TLS is configured, and it supports IPv6 natively. This combination makes Caddy one of the simplest ways to serve HTTP/3 over IPv6 with automatic certificate management.

## HTTP/3 Status in Caddy

Caddy has had HTTP/3 support since version 2 via the `quic-go` library:

```bash
# Check Caddy version and HTTP/3 support
caddy version

# HTTP/3 is enabled by default when HTTPS is configured
# Check experimental flag if needed (older versions):
# experimental_http3 = true (not needed in recent versions)
```

## Basic Caddyfile with HTTP/3 and IPv6

```caddy
# /etc/caddy/Caddyfile

{
    # Global options
    email admin@example.com
    # HTTP/3 is enabled by default
    # Explicit configuration:
    servers {
        protocols h1 h2 h3
    }
}

# Caddy automatically listens on [::]:443 (IPv6) and 0.0.0.0:443 (IPv4)
yourdomain.com {
    # Caddy automatically gets Let's Encrypt cert
    # HTTP/3 is advertised via Alt-Svc header
    root * /var/www/html
    file_server

    # Enable response compression
    encode gzip zstd

    # Log request protocol
    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

## JSON Config with HTTP/3 and IPv6

For more granular control, use Caddy's JSON configuration:

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
          "protocols": ["h1", "h2", "h3"],
          "routes": [
            {
              "match": [{"host": ["yourdomain.com"]}],
              "handle": [
                {
                  "handler": "file_server",
                  "root": "/var/www/html"
                }
              ]
            }
          ],
          "tls_connection_policies": [
            {
              "certificate_selection": {
                "any_tag": ["auto"]
              }
            }
          ]
        }
      }
    },
    "tls": {
      "automation": {
        "policies": [
          {
            "subjects": ["yourdomain.com"],
            "issuers": [{"module": "acme"}]
          }
        ]
      }
    }
  }
}
```

## Testing HTTP/3 with Caddy over IPv6

```bash
# Check Alt-Svc header (indicates HTTP/3 support)
curl -6 -I https://yourdomain.com/ | grep "Alt-Svc"
# Expected: alt-svc: h3=":443"; ma=2592000, h3-29=":443"; ma=2592000

# Test HTTP/3 explicitly
curl --http3 -6 https://yourdomain.com/ -v 2>&1 | grep "HTTP/3\|Version"

# Check QUIC UDP traffic
sudo tcpdump -i eth0 ip6 and udp port 443

# Verify IPv6 is being used
curl -6 -o /dev/null -w "%{remote_ip}\n" https://yourdomain.com/
```

## Caddy with Reverse Proxy and HTTP/3

```caddy
# /etc/caddy/Caddyfile

yourdomain.com {
    # Reverse proxy to backend service
    reverse_proxy /api/* {
        to http://[2001:db8::backend]:8080

        # HTTP/2 to backend (if backend supports it)
        transport http {
            versions 2
        }
    }

    # Static files
    root * /var/www/html
    file_server

    # Caddy handles HTTP/3 negotiation with clients automatically
}
```

## Exposing IPv6-Only Service via Caddy

```caddy
# /etc/caddy/Caddyfile

{
    # Only listen on IPv6
    default_bind [2001:db8::1]
}

yourdomain.com {
    # HTTP/3 on IPv6-only listener
    root * /var/www/html
    file_server
}
```

## Monitoring Caddy HTTP/3 Performance

```bash
# Check Caddy metrics (Prometheus-compatible)
# Enable metrics in Caddyfile:
# servers {
#   metrics
# }

curl http://[::1]:2019/metrics | grep "caddy_http_requests"

# View access logs for protocol distribution
cat /var/log/caddy/access.log | \
  python3 -c "
import sys, json
h3=h2=h1=0
for line in sys.stdin:
    d = json.loads(line)
    proto = d.get('request', {}).get('proto', '')
    if 'HTTP/3' in proto: h3 += 1
    elif 'HTTP/2' in proto: h2 += 1
    else: h1 += 1
print(f'HTTP/3: {h3}, HTTP/2: {h2}, HTTP/1.x: {h1}')
"
```

Caddy's built-in HTTP/3 support combined with automatic HTTPS and native IPv6 binding makes it the most operationally simple way to deploy an HTTP/3-capable web server on IPv6 infrastructure.
