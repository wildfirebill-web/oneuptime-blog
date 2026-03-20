# How to Validate IPv4 Addresses Without Regex in Python Using ipaddress Module

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Validation, ipaddress, Networking, stdlib

Description: Learn how to validate IPv4 addresses in Python without regular expressions using the standard library ipaddress module, which handles all edge cases correctly including leading zeros, ranges, and malformed strings.

## Basic Validation with ipaddress.IPv4Address

```python
import ipaddress

def is_valid_ipv4(s: str) -> bool:
    """Returns True only if s is a strictly valid IPv4 address."""
    try:
        ipaddress.IPv4Address(s)
        return True
    except (ipaddress.AddressValueError, ValueError):
        return False

# Test cases
cases = [
    ("192.168.1.1",     True),
    ("0.0.0.0",         True),
    ("255.255.255.255", True),
    ("256.0.0.1",       False),   # Octet > 255
    ("192.168.1",       False),   # Missing octet
    ("192.168.1.1.1",   False),   # Extra octet
    ("192.168.01.1",    False),   # Leading zero — rejected
    ("::1",             False),   # IPv6
    ("",                False),   # Empty string
    ("localhost",       False),   # Hostname
]

for ip, expected in cases:
    result = is_valid_ipv4(ip)
    status = "PASS" if result == expected else "FAIL"
    print(f"[{status}] {ip!r:<25} -> {result}")
```

## Distinguishing IPv4 from IPv6

```python
import ipaddress

def classify_ip(s: str) -> str:
    """Returns 'ipv4', 'ipv6', or 'invalid'."""
    try:
        addr = ipaddress.ip_address(s)  # accepts both IPv4 and IPv6
        return "ipv4" if isinstance(addr, ipaddress.IPv4Address) else "ipv6"
    except ValueError:
        return "invalid"

print(classify_ip("192.168.1.1"))  # ipv4
print(classify_ip("::1"))          # ipv6
print(classify_ip("not-an-ip"))    # invalid
```

## Checking Address Properties

```python
import ipaddress

def describe_ipv4(s: str) -> dict:
    try:
        addr = ipaddress.IPv4Address(s)
    except ValueError:
        return {"valid": False}

    return {
        "valid":       True,
        "address":     str(addr),
        "is_private":  addr.is_private,
        "is_loopback": addr.is_loopback,
        "is_multicast":addr.is_multicast,
        "is_global":   addr.is_global,
        "packed":      addr.packed.hex(),  # 4-byte big-endian hex
        "integer":     int(addr),
    }

print(describe_ipv4("192.168.1.1"))
# {'valid': True, 'address': '192.168.1.1', 'is_private': True, ...}

print(describe_ipv4("8.8.8.8"))
# {'valid': True, 'address': '8.8.8.8', 'is_private': False, 'is_global': True, ...}
```

## Validating Addresses Inside a CIDR Block

```python
import ipaddress

def is_in_network(ip_str: str, cidr: str) -> bool:
    try:
        ip = ipaddress.IPv4Address(ip_str)
        net = ipaddress.IPv4Network(cidr, strict=False)
        return ip in net
    except ValueError:
        return False

print(is_in_network("192.168.1.50", "192.168.1.0/24"))  # True
print(is_in_network("10.0.0.1",     "192.168.1.0/24"))  # False
print(is_in_network("not-ip",       "192.168.1.0/24"))  # False
```

## Bulk Validation

```python
import ipaddress
from typing import Iterator

def filter_valid_ipv4(addresses: list[str]) -> Iterator[str]:
    """Yield only valid IPv4 addresses from a list."""
    for s in addresses:
        try:
            ipaddress.IPv4Address(s)
            yield s
        except ValueError:
            pass

raw = ["192.168.1.1", "bad", "10.0.0.1", "256.1.1.1", "172.16.0.1"]
valid = list(filter_valid_ipv4(raw))
print(valid)  # ['192.168.1.1', '10.0.0.1', '172.16.0.1']
```

## Conclusion

The `ipaddress.IPv4Address` constructor is the most reliable way to validate IPv4 strings in Python. It correctly rejects leading zeros, out-of-range octets, IPv6 notation, and malformed strings without any regex. Use `ipaddress.ip_address()` when you want to accept both IPv4 and IPv6. Regex remains useful only when you need to extract IP-like strings from freeform text where the `ipaddress` module cannot be applied directly to substrings.
