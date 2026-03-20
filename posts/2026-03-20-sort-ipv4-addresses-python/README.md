# How to Sort a List of IPv4 Addresses in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Sorting, ipaddress, Networking, stdlib

Description: Learn how to sort IPv4 address lists correctly in Python, avoiding the lexicographic pitfall of string sorting by using integer-based comparison with the ipaddress module.

## Correct Sorting with ipaddress.IPv4Address

```python
import ipaddress

ips = [
    "10.0.0.200",
    "10.0.0.9",
    "192.168.1.1",
    "10.0.0.10",
    "172.16.0.1",
    "10.0.0.100",
]

# Use IPv4Address as the sort key — sorts by 32-bit integer value
sorted_ips = sorted(ips, key=ipaddress.IPv4Address)
print(sorted_ips)
# ['10.0.0.9', '10.0.0.10', '10.0.0.100', '10.0.0.200', '172.16.0.1', '192.168.1.1']
```

## Why String Sorting Is Wrong

```python
ips = ["10.0.0.200", "10.0.0.10", "10.0.0.9"]

# String sort — lexicographic, gives wrong order
print(sorted(ips))
# ['10.0.0.10', '10.0.0.200', '10.0.0.9']  ← WRONG (200 before 9?)

# Integer sort via IPv4Address — correct
import ipaddress
print(sorted(ips, key=ipaddress.IPv4Address))
# ['10.0.0.9', '10.0.0.10', '10.0.0.200']  ← CORRECT
```

## Sorting in Reverse (Descending)

```python
import ipaddress

ips = ["192.168.1.1", "10.0.0.1", "172.16.0.1"]
desc = sorted(ips, key=ipaddress.IPv4Address, reverse=True)
print(desc)  # ['192.168.1.1', '172.16.0.1', '10.0.0.1']
```

## Sorting With Mixed Valid/Invalid Strings

```python
import ipaddress

def safe_ip_key(s: str):
    """Return IPv4Address or a fallback value that sorts last."""
    try:
        return (0, ipaddress.IPv4Address(s))
    except ValueError:
        return (1, s)  # invalid strings sort after valid IPs

mixed = ["192.168.1.1", "bad-ip", "10.0.0.1", "not-an-ip", "172.16.0.1"]
print(sorted(mixed, key=safe_ip_key))
# ['10.0.0.1', '172.16.0.1', '192.168.1.1', 'bad-ip', 'not-an-ip']
```

## Sorting IPv4Network Objects

```python
import ipaddress

networks = [
    "192.168.1.0/24",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "10.1.0.0/16",
]

sorted_nets = sorted(networks, key=ipaddress.IPv4Network)
for net in sorted_nets:
    print(net)
# 10.0.0.0/8
# 10.1.0.0/16
# 172.16.0.0/12
# 192.168.0.0/16
# 192.168.1.0/24
```

## Sorting by Multiple Criteria

```python
import ipaddress
from dataclasses import dataclass

@dataclass
class Host:
    name: str
    ip:   str

hosts = [
    Host("gateway", "192.168.1.1"),
    Host("db01",    "10.0.0.5"),
    Host("web01",   "10.0.0.2"),
    Host("cache",   "172.16.0.1"),
]

# Sort by subnet (first 3 octets), then by host address
sorted_hosts = sorted(
    hosts,
    key=lambda h: ipaddress.IPv4Address(h.ip)
)
for h in sorted_hosts:
    print(f"{h.ip:<18} {h.name}")
```

## Conclusion

Always sort IPv4 addresses using `ipaddress.IPv4Address` as the sort key — it compares by integer value and produces numerically correct ordering. String sort treats IP octets as text characters, giving wrong results for mixed-length octets. For lists containing invalid IP strings, use a safe key function that falls back gracefully. The same pattern applies to `IPv4Network` objects, which sort by network address and then prefix length.
