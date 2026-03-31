# How to Handle IPv6 Client Addresses in Serverless Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Serverless, Client Address, Lambda, Function, Rate Limiting

Description: Handle IPv6 client addresses consistently across serverless platforms (AWS Lambda, Azure Functions, GCP Cloud Functions) with common patterns for extraction, validation, and rate limiting.

## Introduction

Serverless functions receive client IP addresses via HTTP headers, but the exact header name and format varies between platforms. IPv6 addresses require special handling for bracket notation, IPv4-mapped representation, and /64-based rate limiting. This post provides platform-agnostic patterns.

## Universal IPv6 Client IP Extractor

```python
def extract_client_ip(headers: dict) -> str:
    """
    Extract the real client IP from serverless function headers.
    Works across AWS Lambda, Azure Functions, GCP Cloud Functions, Vercel.
    """
    # Priority order for headers
    candidate_headers = [
        "x-forwarded-for",     # Most common
        "x-real-ip",           # Nginx proxy
        "cf-connecting-ip",    # Cloudflare
        "x-nf-client-connection-ip",  # Netlify
        "fastly-client-ip",    # Fastly CDN
        "true-client-ip",      # Akamai/Cloudflare Enterprise
        "x-client-ip",         # Some proxies
    ]

    # Normalize header names to lowercase
    headers_lower = {k.lower(): v for k, v in headers.items()}

    for header in candidate_headers:
        value = headers_lower.get(header, "")
        if value:
            # x-forwarded-for may have multiple IPs: "client, proxy1, proxy2"
            ip = value.split(",")[0].strip()
            # Remove brackets if present: [::1] → ::1
            ip = ip.strip("[]")
            return ip

    return "unknown"

def normalize_ipv6(ip: str) -> str:
    """
    Normalize IPv6 representation.
    Converts IPv4-mapped ::ffff:x.x.x.x to plain IPv4.
    """
    import ipaddress
    try:
        addr = ipaddress.ip_address(ip)
        if isinstance(addr, ipaddress.IPv6Address) and addr.ipv4_mapped:
            return str(addr.ipv4_mapped)
        return str(addr)  # Returns compressed IPv6 form
    except ValueError:
        return ip
```

## AWS Lambda Handler

```python
# AWS Lambda with IPv6 client detection

import json
from typing import Any

def lambda_handler(event: dict, context: Any) -> dict:
    headers = event.get("headers", {}) or {}
    client_ip = extract_client_ip(headers)
    client_ip = normalize_ipv6(client_ip)

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "clientIp": client_ip,
            "isIPv6": ":" in client_ip,
        }),
    }
```

## Azure Functions Handler

```python
# Azure Functions with IPv6
import azure.functions as func
import json

def main(req: func.HttpRequest) -> func.HttpResponse:
    headers = dict(req.headers)
    client_ip = extract_client_ip(headers)
    client_ip = normalize_ipv6(client_ip)

    return func.HttpResponse(
        json.dumps({"clientIp": client_ip, "isIPv6": ":" in client_ip}),
        status_code=200,
        mimetype="application/json",
    )
```

## GCP Cloud Functions Handler

```python
# GCP Cloud Functions with IPv6
from flask import Request, jsonify

def handle_request(request: Request):
    headers = dict(request.headers)
    client_ip = extract_client_ip(headers)
    client_ip = normalize_ipv6(client_ip)

    return jsonify({
        "clientIp": client_ip,
        "isIPv6": ":" in client_ip,
    })
```

## IPv6 Rate Limiting (Common Pattern)

```python
import time
from collections import defaultdict
import ipaddress

# In-memory store (use Redis for distributed functions)
rate_store: dict = defaultdict(lambda: {"count": 0, "reset": 0})

def get_rate_key(ip: str, window_seconds: int = 60) -> str:
    """
    Get rate limiting key. Uses /64 subnet for IPv6.
    """
    try:
        addr = ipaddress.ip_address(ip)
        if isinstance(addr, ipaddress.IPv6Address):
            # /64 prefix for IPv6 rate limiting
            prefix = addr.exploded.split(":")
            return ":".join(prefix[:4]) + "::/64"
    except ValueError:
        pass
    return ip

def is_rate_limited(ip: str, limit: int = 100, window: int = 60) -> bool:
    """Check if IP is rate limited. Returns True if over limit."""
    key = get_rate_key(ip, window)
    now = time.time()
    entry = rate_store[key]

    if now > entry["reset"]:
        entry["count"] = 0
        entry["reset"] = now + window

    entry["count"] += 1
    return entry["count"] > limit
```

## Logging IPv6 Addresses Safely

```python
import re
import hashlib

def anonymize_ipv6(ip: str) -> str:
    """
    Anonymize IPv6 for GDPR-compliant logging.
    Masks the interface identifier (last 64 bits).
    """
    try:
        import ipaddress
        addr = ipaddress.IPv6Address(ip)
        # Zero out the last 64 bits (interface ID)
        addr_int = int(addr)
        masked_int = addr_int & (0xFFFFFFFFFFFFFFFF << 64)
        return str(ipaddress.IPv6Address(masked_int)) + "/64"
    except ValueError:
        return ip  # Return as-is if not valid IPv6
```

## Conclusion

Consistent IPv6 client address handling in serverless functions requires a priority-ordered header extraction function, normalization of IPv4-mapped addresses, and /64-based rate limiting. The patterns in this post work across AWS Lambda, Azure Functions, and GCP Cloud Functions. Store rate limit counters in Redis for distributed serverless deployments. Monitor function behavior with OneUptime using synthetic probes from IPv6 networks.
