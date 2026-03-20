# How to Handle IPv6 in Load Balancer X-Forwarded-For Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Load Balancer, X-Forwarded-For, Nginx, Proxy, Web Development

Description: Correctly parse and use IPv6 addresses from X-Forwarded-For headers in applications and reverse proxies, handling bracket notation and IPv4-mapped addresses.

## Introduction

The `X-Forwarded-For` (XFF) header is added by load balancers and proxies to preserve the original client IP. When clients connect over IPv6, their address appears in XFF either as a plain IPv6 address or occasionally with brackets. Applications must correctly extract IPv6 from XFF headers to avoid logging wrong IPs or applying incorrect rate limits.

## X-Forwarded-For Header Format with IPv6

The XFF header format for a chain of proxies:

```
X-Forwarded-For: <client>, <proxy1>, <proxy2>

Examples with IPv6:
X-Forwarded-For: 2001:db8::1, 10.0.0.5, 10.0.0.1
X-Forwarded-For: [2001:db8::1]:12345, 10.0.0.5  # With port (rare)
X-Forwarded-For: ::ffff:192.168.1.5, 10.0.0.1    # IPv4-mapped
X-Forwarded-For: 2001:db8::1                      # Single client
```

## Nginx: Forwarding IPv6 Client IPs

Configure Nginx to forward the real client IPv6 address to upstream applications:

```nginx
# /etc/nginx/conf.d/proxy.conf
server {
    listen 80;
    listen [::]:80;
    server_name example.com;

    location / {
        # Trust our known proxy IPs (adjust to your infrastructure)
        set_real_ip_from 10.0.0.0/8;
        set_real_ip_from 172.16.0.0/12;
        set_real_ip_from 192.168.0.0/16;
        set_real_ip_from 2001:db8:proxy::/48;

        # Use X-Real-IP or XFF for real IP
        real_ip_header X-Forwarded-For;
        real_ip_recursive on;

        # Forward client IP in XFF
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;

        proxy_pass http://backend;
    }
}
```

## Python: Parsing IPv6 from X-Forwarded-For

```python
import ipaddress
import re
from typing import Optional

# Trusted proxy networks that are allowed to set XFF
TRUSTED_PROXIES = [
    ipaddress.ip_network('127.0.0.0/8'),
    ipaddress.ip_network('::1/128'),
    ipaddress.ip_network('10.0.0.0/8'),
    ipaddress.ip_network('172.16.0.0/12'),
    ipaddress.ip_network('192.168.0.0/16'),
    ipaddress.ip_network('2001:db8:proxy::/48'),
]

def is_trusted_proxy(ip_str: str) -> bool:
    """Check if an IP belongs to a trusted proxy range."""
    try:
        addr = ipaddress.ip_address(ip_str.split('%')[0])
        return any(addr in network for network in TRUSTED_PROXIES)
    except ValueError:
        return False

def extract_client_ip(request_headers: dict, remote_addr: str) -> str:
    """
    Extract the real client IP from XFF header, walking back through proxies.
    Returns the leftmost non-trusted IP, or falls back to remote_addr.
    """
    xff = request_headers.get('X-Forwarded-For', '')
    if not xff:
        return normalize_ip(remote_addr)

    # Split XFF and clean each IP
    ips = [clean_xff_entry(ip.strip()) for ip in xff.split(',')]

    # Walk from right to left, skip trusted proxies
    # The client IP is the rightmost non-trusted entry
    # (or leftmost entry if all are trusted)
    last_non_trusted = ips[0]
    for ip in reversed(ips):
        if not is_trusted_proxy(ip):
            last_non_trusted = ip
            break

    return normalize_ip(last_non_trusted)

def clean_xff_entry(entry: str) -> str:
    """
    Clean a single IP entry from XFF, handling:
    - Brackets: [2001:db8::1]
    - Port suffix: [2001:db8::1]:12345 or 192.168.1.1:8080
    - Zone IDs: 2001:db8::1%eth0
    """
    # Remove surrounding brackets
    if entry.startswith('['):
        bracket_end = entry.find(']')
        if bracket_end != -1:
            entry = entry[1:bracket_end]

    # Strip port from IPv4 (x.x.x.x:port)
    if ':' not in entry.split('%')[0].replace(':', '', entry.count(':') - 1) if entry.count(':') == 1 else False:
        entry = entry.rsplit(':', 1)[0]

    # Strip zone ID
    return entry.split('%')[0].strip()

def normalize_ip(ip_str: str) -> str:
    """Normalize IP to compressed form, converting IPv4-mapped to IPv4."""
    clean = ip_str.strip().split('%')[0]
    try:
        addr = ipaddress.ip_address(clean)
        if isinstance(addr, ipaddress.IPv6Address) and addr.ipv4_mapped:
            return str(addr.ipv4_mapped)
        return str(addr)
    except ValueError:
        return clean

# Test
headers = {'X-Forwarded-For': '2001:db8::1, 10.0.0.5, 10.0.0.1'}
client_ip = extract_client_ip(headers, '10.0.0.1')
print(f"Client IP: {client_ip}")  # 2001:db8::1
```

## Node.js: Express Real IP Extraction

```javascript
const net = require('net');

const TRUSTED_PROXIES = [
  { network: '::1', bits: 128 },
  { network: '127.0.0.1', bits: 8 },
  { network: '10.0.0.0', bits: 8 },
];

function extractClientIP(req) {
  const xff = req.headers['x-forwarded-for'] || '';
  const remoteAddr = (req.socket.remoteAddress || '').replace(/^::ffff:/, '');

  if (!xff) return remoteAddr;

  const ips = xff.split(',').map(ip => {
    return ip.trim()
      .replace(/^\[/, '').replace(/\].*$/, '')  // Remove brackets+port
      .split('%')[0]                              // Remove zone ID
      .replace(/^::ffff:/, '');                  // Remove IPv4-mapped prefix
  });

  // Return the leftmost non-internal IP
  for (const ip of ips) {
    if (net.isIP(ip) && !isInternalIP(ip)) {
      return ip;
    }
  }

  return ips[0] || remoteAddr;
}

function isInternalIP(ip) {
  // Simple check for private/reserved ranges
  if (ip === '::1' || ip === '127.0.0.1') return true;
  if (ip.startsWith('10.') || ip.startsWith('192.168.') || ip.startsWith('172.')) return true;
  if (ip.startsWith('fd') || ip.startsWith('fe80')) return true;
  return false;
}

// Express middleware
app.use((req, res, next) => {
  req.clientIP = extractClientIP(req);
  next();
});
```

## Conclusion

IPv6 addresses in X-Forwarded-For headers appear without brackets in most implementations but must be handled with bracket notation support for edge cases. Always validate that XFF was set by a trusted proxy before trusting it, walk the XFF chain from right to left to find the first non-trusted IP, and normalize IPv4-mapped IPv6 addresses (`::ffff:x.x.x.x`) to their IPv4 equivalents.
