# How to Calculate IPv6 Subnets in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv6, Subnetting, ipaddress, Network Automation, IPAM

Description: Calculate IPv6 subnets, split networks, find available prefixes, and generate allocation plans using Python's ipaddress module.

## Splitting a Network into Subnets

Divide a large IPv6 block into smaller prefixes:

```python
import ipaddress

def split_network(network_str: str, new_prefix: int) -> list[ipaddress.IPv6Network]:
    """Split an IPv6 network into subnets of a given prefix length."""
    parent = ipaddress.IPv6Network(network_str)

    if new_prefix <= parent.prefixlen:
        raise ValueError(f"New prefix /{new_prefix} must be larger than /{parent.prefixlen}")

    return list(parent.subnets(new_prefix=new_prefix))

# Split a /48 into /56 subnets (for rack allocations)
rack_subnets = split_network("2001:db8:1::/48", new_prefix=56)
print(f"Total /56 subnets from /48: {len(rack_subnets)}")  # 256

# Show first 5
for i, subnet in enumerate(rack_subnets[:5]):
    print(f"Rack {i+1}: {subnet}")
```

## Finding the Nth Subnet

Get a specific subnet by index without generating all of them:

```python
import ipaddress

def get_nth_subnet(parent_str: str, prefix: int, n: int) -> ipaddress.IPv6Network:
    """
    Get the nth subnet of a given prefix length from a parent network.
    More efficient than generating all subnets.
    """
    parent = ipaddress.IPv6Network(parent_str)
    parent_int = int(parent.network_address)

    # Calculate the size of each subnet
    bits_diff = prefix - parent.prefixlen
    subnet_size = 2 ** (128 - prefix)

    # Calculate the nth subnet's network address
    nth_network_int = parent_int + (n * subnet_size)
    nth_network = ipaddress.IPv6Address(nth_network_int)

    return ipaddress.IPv6Network(f"{nth_network}/{prefix}")

# Get the 100th /64 subnet from a /56
subnet = get_nth_subnet("2001:db8:1::/56", prefix=64, n=100)
print(f"100th /64: {subnet}")   # 2001:db8:1:6400::/64
```

## Calculating Available Subnets in a Range

Find unused subnets given a list of already-allocated ones:

```python
import ipaddress
from typing import Generator

def find_available_subnets(
    parent_str: str,
    prefix: int,
    allocated: list[str],
    count: int = 10
) -> Generator[ipaddress.IPv6Network, None, None]:
    """
    Find available subnets within a parent network,
    excluding already-allocated ones.
    """
    parent = ipaddress.IPv6Network(parent_str)
    allocated_nets = set(
        ipaddress.IPv6Network(a, strict=False) for a in allocated
    )
    found = 0

    for subnet in parent.subnets(new_prefix=prefix):
        if subnet not in allocated_nets:
            yield subnet
            found += 1
            if found >= count:
                return

# Example: Find available /48s in a /40 block
allocated = [
    "2001:db8:1::/48",
    "2001:db8:2::/48",
    "2001:db8:5::/48",
]

available = list(find_available_subnets("2001:db8::/40", 48, allocated, count=5))
for net in available:
    print(f"Available: {net}")
```

## Subnet Summary Table Generator

Generate a complete subnet allocation table:

```python
import ipaddress

def generate_allocation_table(parent_str: str, subnet_prefix: int, count: int) -> list[dict]:
    """Generate an allocation table for subnets."""
    parent = ipaddress.IPv6Network(parent_str)
    allocations = []

    for i, subnet in enumerate(parent.subnets(new_prefix=subnet_prefix)):
        if i >= count:
            break
        allocations.append({
            "index": i,
            "prefix": str(subnet),
            "first_host": str(list(subnet.hosts())[0]) if subnet.prefixlen <= 126 else str(subnet.network_address + 1),
            "last_host": str(list(subnet.hosts())[-1]) if subnet.prefixlen <= 126 else str(subnet.network_address + 2),
            "num_addresses": subnet.num_addresses,
        })

    return allocations

# Generate first 5 /64 subnets from a /48
table = generate_allocation_table("2001:db8:1::/48", 64, count=5)
for row in table:
    print(f"Subnet {row['index']:3}: {row['prefix']:30} ({row['num_addresses']:,} addresses)")
```

## Checking Subnet Containment

```python
import ipaddress

def is_subnet_of(child_str: str, parent_str: str) -> bool:
    """Check if child is a subnet of parent."""
    child = ipaddress.IPv6Network(child_str, strict=False)
    parent = ipaddress.IPv6Network(parent_str, strict=False)
    return child.subnet_of(parent)

# Tests
print(is_subnet_of("2001:db8:1::/64", "2001:db8::/32"))   # True
print(is_subnet_of("2001:db8:1::/64", "2001:db8:2::/48")) # False
```

## Conclusion

Python's `ipaddress` module provides all the tools needed for IPv6 subnet calculation. The `subnets()` method generates sub-prefixes, while integer arithmetic on network addresses enables efficient nth-subnet calculation without iterating through all possibilities. These building blocks form the foundation of IPAM automation scripts and network planning tools.
