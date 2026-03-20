# How to Understand Unique-Local Addresses (fc00::/7) - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Unique-Local, ULA, Fc00, RFC 4193, Private Networking

Description: Understand IPv6 Unique-Local Addresses (fc00::/7), how they function as the IPv6 equivalent of RFC 1918 private addresses, and how to generate and use them.

## Introduction

Unique-Local Addresses (ULA), defined in RFC 4193, are the IPv6 equivalent of IPv4 private addresses (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16). They provide stable, globally unique (within an organization) IPv6 addresses that are not intended for internet routing.

## ULA Structure

```text
fc00::/7 covers two sub-ranges:
  fc00::/8 - Centrally assigned (not yet defined)
  fd00::/8 - Locally generated (most common - use this)

Format for fd00::/8:
  fd  XX:XXXX:XXXX  ::/48  (global ID, randomly chosen)
  └─ fd prefix      └─ subnet
  
Full structure:
  | fd (8) | Global ID (40) | Subnet (16) | Interface ID (64) |
```

## Generating a ULA Prefix

```python
import os
import hashlib
import ipaddress
import time

def generate_ula_prefix() -> str:
    """
    Generate a random ULA /48 prefix per RFC 4193 §3.2.2.
    Uses SHA-1 of EUI-64 + current time.
    """
    # Step 1: Get 64-bit time value
    time_bytes = int(time.time() * 1e7).to_bytes(8, 'big')

    # Step 2: Get or generate a 64-bit system identifier (MAC-based)
    system_id = os.urandom(8)  # Random fallback

    # Step 3: Hash the concatenation
    hash_input = time_bytes + system_id
    h = hashlib.sha1(hash_input).digest()

    # Step 4: Take the last 40 bits of the hash as Global ID
    global_id = int.from_bytes(h[-5:], 'big')

    # Step 5: Build the /48 prefix with fd00::/8 + global_id
    prefix_int = (0xfd << 120) | (global_id << 80)
    prefix_addr = ipaddress.IPv6Address(prefix_int)

    return f"{prefix_addr}/48"

# Example

ula_prefix = generate_ula_prefix()
print(f"Generated ULA /48: {ula_prefix}")
# Example: fd1a:2b3c:4d5e::/48
```

## Allocating Subnets from a ULA /48

```bash
# Given ULA prefix: fd1a:2b3c:4d5e::/48
# Allocate /64 subnets for different VLANs

# VLAN 10 (Servers)
ip -6 addr add fd1a:2b3c:4d5e:10::1/64 dev eth0

# VLAN 20 (Workstations)
ip -6 addr add fd1a:2b3c:4d5e:20::1/64 dev eth1

# VLAN 30 (IoT devices)
ip -6 addr add fd1a:2b3c:4d5e:30::1/64 dev eth2

# Configure RA to advertise these prefixes
# Using radvd or systemd-networkd
```

## ULA vs Global Unicast

| Property | ULA (fd::/8) | Global Unicast (2000::/3) |
|---|---|---|
| Internet routable | No | Yes |
| Stability | Stable (no ISP dependency) | Changes if ISP changes |
| Uniqueness | High probability globally unique | Guaranteed unique |
| NAT needed | Optional (for internet access) | Not needed |
| Privacy | Good (not externally visible) | Lower |

## Filtering ULA at Network Boundaries

```bash
# ULA should never be routed to the internet
ip6tables -A FORWARD \
  -s fc00::/7 \
  -o eth-external \
  -j DROP

# BGP: filter ULA from being advertised externally
# route-map FILTER-ULA
#   deny 10
#     match ipv6 address prefix-list ULA-PREFIXES
#   permit 20
```

## Conclusion

ULA addresses provide stable private IPv6 addressing that won't change when you change ISPs. Generate a random /48 per RFC 4193, allocate /64 subnets for each network segment, and filter ULA at internet boundaries. Use OneUptime to monitor internal services using ULA addresses for infrastructure health.
