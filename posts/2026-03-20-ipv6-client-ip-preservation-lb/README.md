# How to Handle IPv6 Client IP Preservation in Load Balancers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Load Balancer, Client IP, X-Forwarded-For, Proxy Protocol, Logging

Description: A guide to preserving and forwarding the real IPv6 client IP address through load balancers to backend servers for logging, security, and geo-based decisions.

When a load balancer accepts an IPv6 client connection and proxies it to a backend, the backend typically sees the load balancer's IP as the source - not the client's IPv6 address. Preserving the original client IPv6 address is critical for logging, rate limiting, geo-blocking, and security monitoring.

## Methods for IPv6 Client IP Preservation

| Method | Works With | Header |
|---|---|---|
| X-Forwarded-For | HTTP only | `X-Forwarded-For: 2001:db8::client` |
| X-Real-IP | HTTP only | `X-Real-IP: 2001:db8::client` |
| Proxy Protocol v2 | TCP/HTTP | Binary header (layer 3/4) |
| Direct Server Return | TCP | No forwarding needed |

## HAProxy: X-Forwarded-For for IPv6

```text
# /etc/haproxy/haproxy.cfg

frontend ipv6_frontend
    bind [::]:443 ssl crt /etc/ssl/cert.pem

    # Add X-Forwarded-For header with real client IPv6
    option forwardfor

    default_backend web_backends

backend web_backends
    # Set X-Real-IP from XFF header
    http-request set-header X-Real-IP %[req.hdr(X-Forwarded-For)]
    server web1 [2001:db8::web1]:8080 check
```

The `X-Forwarded-For` header for an IPv6 client looks like:
```text
X-Forwarded-For: 2001:db8::client, 2001:db8::lb
```

## Proxy Protocol v2 for IPv6

Proxy Protocol v2 works at Layer 4, carrying the original source and destination addresses before the actual connection data:

```text
# HAProxy sender (front-end LB)

frontend ipv6_frontend
    bind [::]:443 ssl crt /etc/ssl/cert.pem
    default_backend nginx_backends

backend nginx_backends
    # Send Proxy Protocol to backends
    server nginx1 [2001:db8::nginx1]:8080 check send-proxy-v2
```

```nginx
# nginx receiver: accept Proxy Protocol
server {
    listen [::]:8080 proxy_protocol;

    # Get real client IPv6 from Proxy Protocol
    set_real_ip_from [2001:db8::lb]/128;
    real_ip_header proxy_protocol;

    # Log shows real client IPv6
    log_format with_real_ip '$proxy_protocol_addr - $request';
}
```

## nginx: Preserve IPv6 with X-Forwarded-For

```nginx
http {
    # Trust the load balancer's IPv6 address range
    set_real_ip_from 2001:db8:lb::/64;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    server {
        listen [::]:80;
        listen [::]:443 ssl;

        location / {
            proxy_pass http://backends;

            # Forward original client IPv6
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Real-IP $remote_addr;

            # For IPv6 in URLs, keep the bracket notation intact
            proxy_set_header X-Original-IPv6 $remote_addr;
        }
    }
}
```

## Application: Reading IPv6 Client IP

### Python (Flask)

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def index():
    # Get real client IPv6 from X-Forwarded-For
    xff = request.headers.get('X-Forwarded-For')
    if xff:
        # First IP in the chain is the original client
        client_ip = xff.split(',')[0].strip()
    else:
        client_ip = request.remote_addr

    return f"Your IPv6: {client_ip}"
```

### Node.js

```javascript
app.set('trust proxy', true);

app.get('/', (req, res) => {
  const clientIP = req.ip;  // Automatically resolves XFF with trust proxy
  res.send(`Your IPv6: ${clientIP}`);
});
```

## AWS ALB: IPv6 in X-Forwarded-For

AWS ALB automatically adds the client IPv6 to X-Forwarded-For:

```bash
# ALB access log format shows client IPv6
# 2001:db8::client - example.com "GET / HTTP/1.1" 200 1234 ...

# Application receives:
# X-Forwarded-For: 2001:db8::client
# X-Forwarded-Proto: https
```

## Cloudflare: Preserving IPv6 Through CDN

Cloudflare adds the original IPv6 in `CF-Connecting-IP`:

```nginx
# Trust Cloudflare and use CF-Connecting-IP
# List of Cloudflare IPv6 ranges: https://www.cloudflare.com/ips-v6
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
# ... (add all Cloudflare IPv6 ranges)
real_ip_header CF-Connecting-IP;
```

## Logging IPv6 Client IPs

```nginx
# nginx log format that captures real IPv6 client
log_format ipv6_aware '$realip_remote_addr [$time_local] '
                      '"$request" $status $body_bytes_sent '
                      '"$http_referer" "$http_user_agent"';

access_log /var/log/nginx/access.log ipv6_aware;
```

IPv6 client IP preservation requires configuring both the load balancer to forward the original address and the backend to trust and read the forwarded address - ensuring accurate logging and security controls for IPv6 traffic.
