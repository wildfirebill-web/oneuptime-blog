# How to Parse IPv4 CIDR Notation in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, CIDR, Ipaddress, Networking, Subnetting

Description: Learn how to parse and work with IPv4 CIDR notation in Python using the ipaddress standard library module to extract network addresses, host ranges, subnet masks, and perform membership checks.

## Parsing a CIDR Block

```python
import ipaddress

# strict=False allows host bits to be set (e.g., 192.168.1.50/24 → 192.168.1.0/24)

net = ipaddress.IPv4Network("192.168.1.0/24", strict=True)

print(net.network_address)   # 192.168.1.0
print(net.broadcast_address) # 192.168.1.255
print(net.netmask)           # 255.255.255.0
print(net.prefixlen)         # 24
print(net.num_addresses)     # 256
```

## Iterating Hosts

```python
import ipaddress

net = ipaddress.IPv4Network("10.0.0.0/29")

# .hosts() excludes network and broadcast addresses
hosts = list(net.hosts())
print(f"Usable hosts ({len(hosts)}): {hosts[0]} – {hosts[-1]}")
# Usable hosts (6): 10.0.0.1 – 10.0.0.6

# Iterate all 8 addresses including network and broadcast
for addr in net:
    print(addr)
```

## Host Membership Check

```python
import ipaddress

def is_in_subnet(ip: str, cidr: str) -> bool:
    try:
        return ipaddress.IPv4Address(ip) in ipaddress.IPv4Network(cidr, strict=False)
    except ValueError:
        return False

print(is_in_subnet("192.168.1.50",  "192.168.1.0/24"))  # True
print(is_in_subnet("192.168.2.1",   "192.168.1.0/24"))  # False
print(is_in_subnet("10.0.0.1",      "10.0.0.0/8"))      # True
```

## Parsing with Host Bits Set

```python
import ipaddress

# strict=False silently masks off the host bits
cidr_str = "192.168.1.50/24"  # host bits set

net_strict = ipaddress.IPv4Network(cidr_str, strict=False)
print(net_strict)  # 192.168.1.0/24

# Preserve the host address as an IPv4Interface
iface = ipaddress.IPv4Interface(cidr_str)
print(iface.ip)      # 192.168.1.50   (host address)
print(iface.network) # 192.168.1.0/24 (network)
```

## Subnetting: Splitting a Network

```python
import ipaddress

parent = ipaddress.IPv4Network("10.0.0.0/24")

# Split /24 into four /26 subnets
subnets = list(parent.subnets(new_prefix=26))
for s in subnets:
    print(f"{s}  hosts: {s.network_address+1} – {s.broadcast_address-1}")
# 10.0.0.0/26   hosts: 10.0.0.1  – 10.0.0.62
# 10.0.0.64/26  hosts: 10.0.0.65 – 10.0.0.126
# 10.0.0.128/26 ...
# 10.0.0.192/26 ...
```

## Supernetting: Aggregating Addresses

```python
import ipaddress

# Collapse a list of networks into the minimal set of CIDR blocks
addresses = [
    ipaddress.IPv4Network("192.168.1.0/25"),
    ipaddress.IPv4Network("192.168.1.128/25"),
]
collapsed = list(ipaddress.collapse_addresses(addresses))
print(collapsed)  # [IPv4Network('192.168.1.0/24')]
```

## Conclusion

Python's `ipaddress.IPv4Network` makes CIDR parsing straightforward: construct with a prefix string and `strict=False` to handle addresses with host bits set. Use `.hosts()` for the usable host range, `in` for membership tests, `.subnets()` to split, and `collapse_addresses()` to aggregate. The `IPv4Interface` type is useful when you need to keep both the host address and its network context simultaneously.
