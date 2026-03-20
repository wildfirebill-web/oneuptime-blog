# How to Validate IPv4 Addresses in Python with the ipaddress Module

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Networking, ipaddress, Validation, Programming

Description: Learn how to use Python's built-in ipaddress module to validate IPv4 addresses and CIDR blocks, check address types, and perform network calculations.

## Introduction

Python's built-in `ipaddress` module (introduced in Python 3.3) provides robust tools for working with IPv4 and IPv6 addresses. It handles validation, type checking, and network calculations without any third-party dependencies.

## Basic IPv4 Address Validation

```python
import ipaddress

def is_valid_ipv4(address: str) -> bool:
    try:
        ipaddress.IPv4Address(address)
        return True
    except ipaddress.AddressValueError:
        return False

# Examples
print(is_valid_ipv4("192.168.1.1"))    # True
print(is_valid_ipv4("256.0.0.1"))      # False
print(is_valid_ipv4("192.168.1"))      # False
print(is_valid_ipv4("not-an-ip"))      # False
```

## Validating IPv4 CIDR Blocks

```python
def is_valid_ipv4_cidr(cidr: str) -> bool:
    try:
        ipaddress.IPv4Network(cidr, strict=False)
        return True
    except (ipaddress.AddressValueError, ipaddress.NetmaskValueError, ValueError):
        return False

print(is_valid_ipv4_cidr("10.0.0.0/8"))     # True
print(is_valid_ipv4_cidr("10.0.1.5/24"))    # True (strict=False allows host bits)
print(is_valid_ipv4_cidr("invalid/24"))     # False
```

## Checking Address Types

```python
addr = ipaddress.IPv4Address("192.168.1.1")

print(addr.is_private)     # True
print(addr.is_global)      # False
print(addr.is_loopback)    # False
print(addr.is_multicast)   # False
print(addr.is_reserved)    # False
print(addr.is_link_local)  # False
```

## Checking if an IP is in a Network

```python
network = ipaddress.IPv4Network("10.0.0.0/8")
address = ipaddress.IPv4Address("10.1.2.3")

print(address in network)  # True
```

## Getting Network Information

```python
net = ipaddress.IPv4Network("192.168.1.0/24")

print(net.network_address)    # 192.168.1.0
print(net.broadcast_address)  # 192.168.1.255
print(net.netmask)            # 255.255.255.0
print(net.prefixlen)          # 24
print(net.num_addresses)      # 256
print(list(net.hosts())[:3])  # First 3 host addresses
```

## Iterating Over Hosts

```python
net = ipaddress.IPv4Network("10.0.0.0/30")
for host in net.hosts():
    print(host)
# 10.0.0.1
# 10.0.0.2
```

## Real-World Validation Function

```python
def validate_network_config(ip: str, cidr: str) -> dict:
    result = {"valid": False, "errors": []}

    try:
        addr = ipaddress.IPv4Address(ip)
        net = ipaddress.IPv4Network(cidr, strict=False)

        if addr not in net:
            result["errors"].append(f"{ip} is not in network {cidr}")
        else:
            result["valid"] = True
            result["network"] = str(net)
            result["is_private"] = addr.is_private

    except ValueError as e:
        result["errors"].append(str(e))

    return result

print(validate_network_config("10.0.1.5", "10.0.0.0/16"))
```

## Conclusion

Python's `ipaddress` module provides everything needed for IPv4 validation and network calculations without external dependencies. It handles edge cases correctly and raises clear exceptions for invalid input, making it the best choice for IPv4 validation in Python applications.
