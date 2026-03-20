# How to Use VLSM for Efficient Address Allocation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, VLSM, Subnetting, Networking, Address Allocation, Network Design

Description: Variable Length Subnet Masking (VLSM) allows different subnets within the same address block to use different prefix lengths, allocating exactly the right number of addresses to each segment and...

## What Is VLSM?

VLSM (Variable Length Subnet Masking) enables assigning different mask lengths to different subnets carved from the same parent block. This contrasts with Fixed-Length Subnet Masking (FLSM), which forces all subnets to the same size.

## FLSM vs VLSM Comparison

Suppose you need:
- Segment A: 100 hosts
- Segment B: 50 hosts
- Segment C: 20 hosts
- P2P link: 2 hosts

**FLSM** (forced /24 for all): wastes thousands of addresses
**VLSM**:
- Segment A → /25 (126 hosts)
- Segment B → /26 (62 hosts)
- Segment C → /27 (30 hosts)
- P2P link → /30 (2 hosts)

## Python VLSM Allocator

```python
import ipaddress
import math

def vlsm_allocate(parent: str, segments: list) -> list:
    """
    Allocate subnets using VLSM from a parent block.
    segments: list of (name, required_hosts) tuples
    Subnets must be sorted largest-first for efficient allocation.
    """
    # Sort by required hosts descending (largest first)
    sorted_segs = sorted(segments, key=lambda x: x[1], reverse=True)
    parent_net = ipaddress.IPv4Network(parent, strict=False)

    results = []
    current = parent_net.network_address

    for name, hosts in sorted_segs:
        # Find smallest prefix that fits `hosts` usable addresses
        host_bits = math.ceil(math.log2(hosts + 2))
        prefix = 32 - host_bits
        # Align current to the subnet boundary
        subnet = ipaddress.IPv4Network(f"{current}/{prefix}", strict=False)
        # Check it fits within parent
        if not subnet.subnet_of(parent_net):
            print(f"ERROR: {name} does not fit in parent!")
            break
        results.append((name, subnet))
        # Advance current pointer past this subnet
        next_ip_int = int(subnet.broadcast_address) + 1
        current = ipaddress.IPv4Address(next_ip_int)

    return results

# Example: allocate from 192.168.10.0/24

segments = [
    ("Department-A",  100),
    ("Department-B",   50),
    ("Department-C",   20),
    ("Management",     10),
    ("P2P-Link-1",      2),
    ("P2P-Link-2",      2),
]

allocations = vlsm_allocate("192.168.10.0/24", segments)
print(f"{'Segment':15s}  {'Subnet':20s}  Usable")
for name, subnet in allocations:
    print(f"{name:15s}  {str(subnet):20s}  {subnet.num_addresses - 2}")
```

## VLSM Requirements

- Routing protocol must support classless routing (OSPF, BGP, RIPv2, EIGRP).
- RIPv1 and IGRP do NOT support VLSM.

## Key Takeaways

- VLSM assigns the right-sized subnet to each segment, minimizing wasted addresses.
- Always allocate the largest subnets first to ensure alignment and avoid gaps.
- VLSM requires classless routing protocols that carry prefix lengths in their updates.
- The `ipaddress` module's `subnet_of()` method verifies allocations stay within the parent block.
