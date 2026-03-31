# How to Handle IPv6 Subnet Matching in Web Application ACLs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Subnet Matching, ACL, Python, Web Application, Security

Description: Implement IPv6 subnet matching for web application access control lists, handling CIDR notation, prefix compression, and IPv4-mapped addresses.

## Introduction

Web application ACLs often need to match IPv6 client addresses against allowed/denied subnets. IPv6 subnet matching is more complex than IPv4 because addresses can be represented in multiple equivalent forms, prefixes can be /64 for NDP but /128 for individual hosts, and IPv4-mapped addresses must be handled.

## Python IPv6 Subnet Matcher

```python
import ipaddress
from typing import Union

class IPv6SubnetACL:
    """Access Control List supporting both IPv4 and IPv6 subnets."""

    def __init__(self, allowed: list = None, denied: list = None):
        self.allowed = [ipaddress.ip_network(s, strict=False)
                        for s in (allowed or [])]
        self.denied = [ipaddress.ip_network(s, strict=False)
                       for s in (denied or [])]

    def _normalize(self, addr_str: str) -> ipaddress.IPv6Address:
        """
        Normalize a client address to IPv6.
        Handles ::ffff:x.x.x.x (IPv4-mapped) by extracting IPv4.
        """
        # Strip brackets if present ([::1] → ::1)
        addr_str = addr_str.strip("[]")

        addr = ipaddress.ip_address(addr_str)

        # Convert IPv4-mapped to pure IPv4 for consistent matching
        if isinstance(addr, ipaddress.IPv6Address) and addr.ipv4_mapped:
            return addr.ipv4_mapped

        return addr

    def is_allowed(self, client_addr: str) -> bool:
        """Check if client address is allowed by the ACL."""
        try:
            addr = self._normalize(client_addr)
        except ValueError:
            return False

        # Check deny list first (deny overrides allow)
        for net in self.denied:
            if addr in net:
                return False

        # If allow list is empty, allow all (not in deny)
        if not self.allowed:
            return True

        # Check allow list
        for net in self.allowed:
            if addr in net:
                return True

        return False

# Usage example

acl = IPv6SubnetACL(
    allowed=[
        "2001:db8::/32",        # Internal IPv6 range
        "10.0.0.0/8",           # Internal IPv4 range
        "192.168.0.0/16",       # Home/office IPv4
        "fd00::/8",             # ULA range
    ],
    denied=[
        "2001:db8:dead::/48",   # Specific blocked subnet
    ]
)

test_clients = [
    "2001:db8::1",           # Allowed
    "2001:db8:dead::1",      # Denied (in deny list)
    "::ffff:10.0.0.1",       # Allowed (IPv4-mapped → 10.0.0.1)
    "192.168.1.100",         # Allowed
    "2001:4860:4860::8888",  # Denied (not in allow list)
    "fd00:1::abc",           # Allowed (ULA)
]

for client in test_clients:
    print(f"{client:40s} → {'ALLOWED' if acl.is_allowed(client) else 'DENIED'}")
```

## Django Middleware for IPv6 ACL

```python
# myapp/middleware.py
import ipaddress
from django.http import HttpResponseForbidden

ALLOWED_SUBNETS = [
    ipaddress.ip_network("2001:db8::/32"),
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("fd00::/8"),
]

class IPv6ACLMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        client_ip = self._get_client_ip(request)
        if not self._is_allowed(client_ip):
            return HttpResponseForbidden(f"Access denied for {client_ip}")
        return self.get_response(request)

    def _get_client_ip(self, request) -> str:
        # Check X-Forwarded-For (only trust if behind known proxy)
        xff = request.META.get("HTTP_X_FORWARDED_FOR")
        if xff:
            return xff.split(",")[0].strip()
        return request.META.get("REMOTE_ADDR", "")

    def _is_allowed(self, addr_str: str) -> bool:
        try:
            addr = ipaddress.ip_address(addr_str)
            if isinstance(addr, ipaddress.IPv6Address) and addr.ipv4_mapped:
                addr = addr.ipv4_mapped
            return any(addr in net for net in ALLOWED_SUBNETS)
        except ValueError:
            return False
```

## nginx-style Configuration Parsing

```python
def parse_nginx_allow_deny(rules: list) -> IPv6SubnetACL:
    """
    Parse nginx-style allow/deny rules into an ACL object.
    rules: list of strings like "allow 2001:db8::/32", "deny all"
    """
    allowed = []
    denied = []

    for rule in rules:
        parts = rule.strip().split()
        if len(parts) != 2:
            continue
        action, subnet = parts[0].lower(), parts[1]

        if subnet == "all":
            if action == "deny":
                denied.append("::/0")
                denied.append("0.0.0.0/0")
            continue

        if action == "allow":
            allowed.append(subnet)
        elif action == "deny":
            denied.append(subnet)

    return IPv6SubnetACL(allowed=allowed, denied=denied)

# Parse nginx-style rules
rules = [
    "allow 2001:db8::/32",
    "allow 10.0.0.0/8",
    "deny all",
]
acl = parse_nginx_allow_deny(rules)
```

## Conclusion

IPv6 subnet matching in web ACLs requires normalization of IPv4-mapped addresses, handling multiple IPv6 representations, and clear ordering of allow/deny rules. The Python `ipaddress` module's `addr in network` operator handles all prefix length matching correctly. Use the `IPv6SubnetACL` class as a reusable component and integrate with OneUptime to monitor which IPs are accessing restricted endpoints.
