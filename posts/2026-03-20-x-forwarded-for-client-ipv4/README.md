# How to Configure X-Forwarded-For Headers to Preserve Client IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, X-Forwarded-For, IPv4, Reverse Proxy, Header, Security

Description: Learn how to correctly configure X-Forwarded-For headers in Nginx so that your applications receive the real client IPv4 address instead of the proxy's IP.

## The Problem with Reverse Proxies

When traffic flows through a reverse proxy, the upstream application sees the proxy's IP address as the client. The `X-Forwarded-For` (XFF) header solves this by carrying the original client IP through the proxy chain.

## Setting X-Forwarded-For in Nginx

Configure Nginx to pass the real client IP to upstream:

```nginx
# /etc/nginx/sites-available/app.conf

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # Pass the real client IP in X-Forwarded-For
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # $proxy_add_x_forwarded_for appends $remote_addr to any existing
        # X-Forwarded-For header, building a proper chain for multiple proxies

        # Also pass the original scheme
        proxy_set_header X-Forwarded-Proto $scheme;

        # Pass original host
        proxy_set_header Host $host;
    }
}
```

`$proxy_add_x_forwarded_for` is a built-in Nginx variable that either creates the header with `$remote_addr` or appends it to the existing chain.

## Using $realip_module for Trusted Proxies

For scenarios where you need `$remote_addr` itself to reflect the real client IP (not just a header), use `ngx_http_realip_module`:

```nginx
# /etc/nginx/nginx.conf
http {
    # Trust these proxy IP ranges to set real IP via X-Forwarded-For
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from 172.16.0.0/12;
    set_real_ip_from 192.168.0.0/16;
    set_real_ip_from 127.0.0.1;

    # Use the last entry in X-Forwarded-For that isn't a trusted proxy
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;  # Strip trusted proxy IPs from the chain
}
```

With `real_ip_recursive on`, Nginx walks the XFF chain from right to left and sets `$remote_addr` to the first untrusted IP-the actual client.

## Multi-Proxy Chains

When traffic passes through multiple proxies, the XFF header accumulates IPs:

```text
X-Forwarded-For: client-ip, proxy1-ip, proxy2-ip
```

The leftmost IP is the original client. `real_ip_recursive on` handles this correctly by ignoring known trusted proxy IPs from the right.

## Reading the Client IP in Application Code

After proxy configuration, read the header in your application:

```python
# Flask (Python) example
from flask import Flask, request

app = Flask(__name__)

@app.route("/")
def index():
    # With Nginx real_ip_module, request.remote_addr is the real client IP
    client_ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    # Take only the first IP in case of chain
    client_ip = client_ip.split(",")[0].strip()
    return f"Your IP: {client_ip}"
```

```javascript
// Express.js example
// Set trust proxy to respect X-Forwarded-For from known proxies
app.set('trust proxy', 'loopback, linklocal, uniquelocal');

app.get('/', (req, res) => {
    // req.ip is automatically set to the real client IP
    res.send(`Your IP: ${req.ip}`);
});
```

## Security Considerations

- **Never trust XFF blindly** - clients can spoof the header
- Only trust XFF from known, controlled proxy IPs (use `set_real_ip_from`)
- Use `real_ip_recursive on` to strip forged IPs added before your trusted proxies

## Conclusion

Preserving the real client IP through a reverse proxy requires both Nginx configuration (`proxy_set_header X-Forwarded-For`) and, for full transparency, the `realip_module` with `set_real_ip_from` to define trusted proxies. Always validate XFF only from trusted proxy addresses to prevent IP spoofing.
