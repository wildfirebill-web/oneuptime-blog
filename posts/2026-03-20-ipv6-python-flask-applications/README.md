# How to Handle IPv6 in Python Flask Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Flask, IPv6, Web Development, HTTP, REST API

Description: Configure Flask applications to handle IPv6 connections, extract client IPv6 addresses, and serve on IPv6 interfaces.

## Running Flask on IPv6

By default, Flask listens on `0.0.0.0` (IPv4 only). To enable IPv6:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def index():
    return "Hello from Flask over IPv6!"

if __name__ == "__main__":
    # Listen on all IPv4 and IPv6 interfaces
    app.run(
        host="::",         # :: binds to all IPv6 interfaces
        port=5000,
        debug=True
    )
```

Run with:
```bash
python app.py
# Access via: http://[::1]:5000  (localhost IPv6)

# Or: http://[2001:db8::1]:5000  (global IPv6)
```

## Getting Client IPv6 Address

```python
from flask import Flask, request, jsonify
import ipaddress

app = Flask(__name__)

def get_client_ip() -> str:
    """
    Get the real client IP address, handling:
    - Direct IPv6 connections
    - Proxied connections (X-Forwarded-For)
    - IPv4-mapped IPv6 addresses (::ffff:x.x.x.x)
    """
    # Check for proxy header first
    forwarded_for = request.headers.get("X-Forwarded-For")
    if forwarded_for:
        # Take the first IP in the chain (original client)
        ip_str = forwarded_for.split(",")[0].strip()
    else:
        ip_str = request.remote_addr

    # Normalize IPv4-mapped IPv6 addresses
    try:
        ip = ipaddress.IPv6Address(ip_str)
        if ip.ipv4_mapped:
            return str(ip.ipv4_mapped)
        return str(ip)
    except ValueError:
        return ip_str  # Return as-is for IPv4

@app.route("/api/whoami")
def whoami():
    client_ip = get_client_ip()
    return jsonify({
        "ip": client_ip,
        "version": 6 if ":" in client_ip else 4
    })
```

## Rate Limiting with IPv6

Rate limiting by IPv6 address requires handling /64 prefixes (one user may have many /128 addresses):

```python
from flask import Flask, request
from flask_limiter import Limiter
import ipaddress

app = Flask(__name__)

def get_ipv6_rate_limit_key():
    """
    Use /64 prefix as rate limit key for IPv6 (privacy extensions
    mean one user can have many different /128 addresses).
    For IPv4, use the full address.
    """
    ip = request.remote_addr
    try:
        addr = ipaddress.IPv6Address(ip)
        if not addr.is_private and not addr.is_link_local:
            # Use /64 prefix as key for global IPv6
            net = ipaddress.IPv6Network(f"{ip}/64", strict=False)
            return str(net)
    except ValueError:
        pass
    return ip

limiter = Limiter(
    app=app,
    key_func=get_ipv6_rate_limit_key,
    default_limits=["100 per minute"]
)

@app.route("/api/data")
@limiter.limit("10 per second")
def get_data():
    return jsonify({"data": "value"})
```

## Validating IPv6 Input in Flask Routes

```python
from flask import Flask, request, jsonify
import ipaddress

app = Flask(__name__)

@app.route("/api/device/<address>")
def get_device(address: str):
    """
    Route that accepts an IPv6 address parameter.
    Flask routes don't handle IPv6 directly in URL params.
    """
    # Validate the IPv6 address
    try:
        addr = ipaddress.IPv6Address(address)
    except ValueError:
        return jsonify({"error": f"Invalid IPv6 address: {address}"}), 400

    return jsonify({
        "address": str(addr.compressed),
        "type": "link_local" if addr.is_link_local else "global",
        "expanded": addr.exploded
    })

@app.route("/api/device", methods=["POST"])
def create_device():
    """Create a device with IPv6 address validation."""
    data = request.get_json()

    if not data or "ipv6" not in data:
        return jsonify({"error": "Missing ipv6 field"}), 400

    try:
        addr = ipaddress.IPv6Address(data["ipv6"])
    except ValueError as e:
        return jsonify({"error": str(e)}), 400

    # Process valid address
    return jsonify({
        "status": "created",
        "address": str(addr.compressed)
    }), 201
```

## Flask with Gunicorn for Production IPv6

For production, run Flask with Gunicorn bound to IPv6:

```bash
# Bind to all IPv6 interfaces
gunicorn -b "[::]:8000" app:app

# Bind to specific IPv6 address
gunicorn -b "[2001:db8::1]:8000" app:app

# Dual-stack: bind to both IPv4 and IPv6
gunicorn -b "0.0.0.0:8000" -b "[::]:8000" app:app
```

## Nginx Reverse Proxy for IPv6 Flask

Configure Nginx to proxy IPv6 requests to Flask:

```nginx
server {
    listen [::]:80;         # IPv6
    listen 80;              # IPv4
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Conclusion

Flask handles IPv6 by binding to `::` instead of `0.0.0.0`. Client IPv6 address extraction requires normalizing IPv4-mapped addresses and handling privacy extensions for rate limiting. For production deployments, Gunicorn with `[::]:port` binding and Nginx fronting provides a robust IPv6-capable web service stack.
