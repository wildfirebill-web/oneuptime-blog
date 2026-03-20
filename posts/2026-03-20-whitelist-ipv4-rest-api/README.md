# How to Whitelist IPv4 Addresses for REST API Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: REST API, IPv4, Whitelist, Security, Python, Node.js

Description: Learn how to implement IPv4 address whitelisting for REST API access control in Python and Node.js, with CIDR-based allow lists, middleware patterns, and dynamic whitelist management.

## Python / Flask: CIDR Whitelist Middleware

```python
import ipaddress
from flask import Flask, request, jsonify

app = Flask(__name__)

# Allow individual IPs and CIDR blocks

ALLOWED = [
    ipaddress.IPv4Network("192.168.1.0/24"),
    ipaddress.IPv4Network("10.0.0.0/8"),
    ipaddress.IPv4Address("203.0.113.10"),   # single trusted IP
]

def get_client_ip() -> str:
    xff = request.headers.get("X-Forwarded-For")
    return xff.split(",")[0].strip() if xff else request.remote_addr

def is_allowed(ip: str) -> bool:
    try:
        addr = ipaddress.IPv4Address(ip)
        for entry in ALLOWED:
            if isinstance(entry, ipaddress.IPv4Network):
                if addr in entry:
                    return True
            elif addr == entry:
                return True
        return False
    except ValueError:
        return False

@app.before_request
def check_whitelist():
    ip = get_client_ip()
    if not is_allowed(ip):
        return jsonify(error=f"Access denied for {ip}"), 403

@app.get("/api/data")
def data():
    return jsonify(result="ok")
```

## Python: Route-Level Decorator

```python
from functools import wraps
import ipaddress
from flask import request, jsonify

ADMIN_WHITELIST = [ipaddress.IPv4Network("127.0.0.0/8")]

def ipv4_whitelist(*networks):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            ip = request.remote_addr
            try:
                addr = ipaddress.IPv4Address(ip)
                for net in networks:
                    if addr in ipaddress.IPv4Network(net):
                        return f(*args, **kwargs)
            except ValueError:
                pass
            return jsonify(error="Forbidden"), 403
        return wrapper
    return decorator

@app.get("/admin/status")
@ipv4_whitelist("127.0.0.0/8", "10.0.0.0/8")
def admin_status():
    return jsonify(status="ok")
```

## Node.js / Express: Whitelist Middleware

```javascript
const ipaddr  = require("ipaddr.js");
const express = require("express");

const app = express();
app.set("trust proxy", 1);

const ALLOWED_CIDRS = [
    "192.168.1.0/24",
    "10.0.0.0/8",
    "127.0.0.1/32",
];

function isAllowed(ip) {
    try {
        const addr = ipaddr.process(ip);  // normalises IPv4-mapped IPv6
        for (const cidr of ALLOWED_CIDRS) {
            const [range, bits] = ipaddr.parseCIDR(cidr);
            if (addr.kind() === range.kind() && addr.match(range, bits)) {
                return true;
            }
        }
    } catch (_) {
        // invalid IP
    }
    return false;
}

function ipWhitelist(req, res, next) {
    const ip = req.ip;
    if (isAllowed(ip)) return next();
    res.status(403).json({ error: `Access denied for ${ip}` });
}

app.use("/api/", ipWhitelist);
app.get("/api/data", (req, res) => res.json({ result: "ok" }));
app.listen(3000);
```

## Dynamic Whitelist from Database

```python
import ipaddress
from flask import Flask, request, jsonify
import time

app = Flask(__name__)

# Refresh cache every 60 seconds
_cache = {"data": [], "ts": 0}

def load_whitelist() -> list:
    """Load allowed CIDRs from database (stub)."""
    # Replace with actual DB query
    return ["192.168.0.0/16", "10.0.0.0/8"]

def get_whitelist() -> list[ipaddress.IPv4Network]:
    now = time.monotonic()
    if now - _cache["ts"] > 60:
        raw = load_whitelist()
        _cache["data"] = [ipaddress.IPv4Network(c, strict=False) for c in raw]
        _cache["ts"] = now
    return _cache["data"]

@app.before_request
def check_ip():
    ip = request.remote_addr
    try:
        addr = ipaddress.IPv4Address(ip)
        if any(addr in net for net in get_whitelist()):
            return  # allowed
    except ValueError:
        pass
    return jsonify(error="Forbidden"), 403
```

## Conclusion

Use `ipaddress.IPv4Network` and the `in` operator for CIDR-based allow lists in Python. In Node.js, `ipaddr.js` handles both IPv4 and IPv4-mapped IPv6 addresses. Cache database-loaded whitelists with a short TTL to avoid per-request DB calls. Apply whitelist middleware globally to admin routes or sensitive endpoints only - public API routes typically use rate limiting rather than IP restriction. Always verify the source IP from a trusted connection before acting on forwarded headers.
