# How to Use Python ipaddress Module to Create IPv4 Network Objects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, ipaddress, Networking, CIDR, stdlib

Description: Learn how to create and work with IPv4Network and IPv4Interface objects in Python's ipaddress module for subnet management, address enumeration, and network operations.

## Creating IPv4Network Objects

```python
import ipaddress

# From CIDR string — strict=True (default) requires no host bits set
net = ipaddress.IPv4Network("192.168.1.0/24")
print(net)                     # 192.168.1.0/24
print(net.network_address)     # 192.168.1.0
print(net.broadcast_address)   # 192.168.1.255
print(net.netmask)             # 255.255.255.0
print(net.prefixlen)           # 24
print(net.num_addresses)       # 256

# strict=False: silently masks host bits
net2 = ipaddress.IPv4Network("192.168.1.50/24", strict=False)
print(net2)  # 192.168.1.0/24  (host bits cleared)
```

## IPv4Interface: Address + Network

```python
import ipaddress

# IPv4Interface holds both the host address and the network
iface = ipaddress.IPv4Interface("192.168.1.50/24")
print(iface.ip)       # 192.168.1.50  (host address)
print(iface.network)  # 192.168.1.0/24
print(iface.netmask)  # 255.255.255.0
print(str(iface))     # 192.168.1.50/24
```

## Network Membership and Containment

```python
import ipaddress

net = ipaddress.IPv4Network("10.0.0.0/8")

# Check host membership
print(ipaddress.IPv4Address("10.1.2.3") in net)   # True
print(ipaddress.IPv4Address("192.168.1.1") in net) # False

# Check if one network is a subnet of another
child  = ipaddress.IPv4Network("10.1.0.0/16")
parent = ipaddress.IPv4Network("10.0.0.0/8")
print(child.subnet_of(parent))    # True
print(parent.supernet_of(child))  # True
```

## Subnetting and Supernetting

```python
import ipaddress

net = ipaddress.IPv4Network("10.0.0.0/24")

# Split into smaller subnets
for subnet in net.subnets(new_prefix=26):
    print(subnet)
# 10.0.0.0/26
# 10.0.0.64/26
# 10.0.0.128/26
# 10.0.0.192/26

# Get the containing supernet
print(net.supernet())           # 10.0.0.0/23
print(net.supernet(prefixlen_diff=2))  # 10.0.0.0/22

# Collapse a list of networks
blocks = [ipaddress.IPv4Network("10.0.0.0/25"), ipaddress.IPv4Network("10.0.0.128/25")]
print(list(ipaddress.collapse_addresses(blocks)))  # [IPv4Network('10.0.0.0/24')]
```

## Comparing and Sorting Networks

```python
import ipaddress

networks = [
    ipaddress.IPv4Network("192.168.1.0/24"),
    ipaddress.IPv4Network("10.0.0.0/8"),
    ipaddress.IPv4Network("172.16.0.0/12"),
    ipaddress.IPv4Network("10.1.0.0/16"),
]

# Networks are ordered by (network_address, prefix_length)
for net in sorted(networks):
    print(net)
# 10.0.0.0/8
# 10.1.0.0/16
# 172.16.0.0/12
# 192.168.1.0/24
```

## Practical: Build a Simple IP Allocator

```python
import ipaddress
from typing import Iterator

class SubnetAllocator:
    def __init__(self, cidr: str):
        self.pool = ipaddress.IPv4Network(cidr, strict=False)
        self.allocated: list[ipaddress.IPv4Network] = []

    def allocate(self, prefix: int) -> ipaddress.IPv4Network:
        for candidate in self.pool.subnets(new_prefix=prefix):
            if not any(candidate.overlaps(a) for a in self.allocated):
                self.allocated.append(candidate)
                return candidate
        raise RuntimeError("No available subnet")

alloc = SubnetAllocator("10.0.0.0/16")
a = alloc.allocate(24)
b = alloc.allocate(24)
c = alloc.allocate(25)
print(a, b, c)
# 10.0.0.0/24  10.0.1.0/24  10.0.2.0/25
```

## Conclusion

`IPv4Network` is the go-to class for working with network blocks: membership tests with `in`, subnetting with `.subnets()`, and aggregation with `collapse_addresses()`. Use `IPv4Interface` when you need to associate a host address with its network context (e.g., for interface configuration). Both classes are sortable and support comparison operators, making them suitable for use as dictionary keys and in sorted data structures.
