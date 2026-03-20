# How to Understand Unique-Local Addresses (fc00::/7)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Unique-Local, Fc00, Fd00, RFC 4193, Private, Networking

Description: Understand IPv6 Unique-Local Addresses (ULA) in the fc00::/7 range (RFC 4193), their structure, when to use them, and how they differ from IPv4 private addresses.

## Introduction

Unique-Local Addresses (ULA) are the IPv6 equivalent of IPv4 private addresses (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16). They are defined in RFC 4193 and occupy the `fc00::/7` block. ULAs are locally routed but not globally routable, making them suitable for internal networks that do not require internet connectivity.

## ULA Address Structure

```yaml
 | 7 bits  | 1 bit |  40 bits   |  16 bits  |  64 bits    |
 +---------+-------+------------+-----------+-------------+
 | 1111110 |   L   |  Global ID | Subnet ID | Interface ID|
 +---------+-------+------------+-----------+-------------+
 | fc00::/7                     |

 L=0: fc00::/8 - Reserved (not currently assigned)
 L=1: fd00::/8 - Locally assigned (the one you use)

 Most ULA addresses start with fd:
   fd00::/8 = Locally assigned ULA

 Global ID: 40-bit pseudo-random identifier (RFC 4193 §3.2.2)
   Randomly generated per organization to ensure uniqueness
   Even though not globally routed, uniqueness prevents conflicts
   when networks are merged
```

## Generating a ULA Prefix (RFC 4193 Algorithm)

```python
import hashlib
import time
import uuid
import ipaddress

def generate_ula_prefix() -> str:
    """
    Generate a random ULA /48 prefix per RFC 4193 §3.2.2.
    Uses SHA-1(NTP timestamp || EUI-64) → 40-bit Global ID
    """
    # Combine current time and a machine identifier
    ntp_time = int(time.time() * 2**32).to_bytes(8, 'big')
    machine_id = uuid.getnode().to_bytes(8, 'big')  # MAC-based

    # SHA-1 hash
    digest = hashlib.sha1(ntp_time + machine_id).digest()

    # Take last 40 bits (5 bytes) as Global ID
    global_id_bytes = digest[-5:]
    global_id_int = int.from_bytes(global_id_bytes, 'big')

    # Construct fd00::/8 + global_id/40 = /48 prefix
    ula_int = (0xfd << 120) | (global_id_int << 80)
    ula_addr = ipaddress.IPv6Address(ula_int)
    prefix = f"{ula_addr}/48"
    return prefix

prefix = generate_ula_prefix()
print(f"Generated ULA prefix: {prefix}")
# Example: fd3a:b2c1:d4e5::/48

```

## Subnet Planning with ULA

```python
import ipaddress

# Assume organization has fd3a:b2c1:d4e5::/48
org_prefix = ipaddress.IPv6Network("fd3a:b2c1:d4e5::/48")

# Sub-allocate /64s for each VLAN
subnets = list(org_prefix.subnets(prefixlen_diff=16))  # 65536 /64s

print("First 10 /64 subnets:")
for net in subnets[:10]:
    print(f"  {net}")

# Assign specific VLANs
vlans = {
    "servers":      subnets[10],   # fd3a:b2c1:d4e5:a::/64
    "workstations": subnets[20],   # fd3a:b2c1:d4e5:14::/64
    "iot":          subnets[30],   # fd3a:b2c1:d4e5:1e::/64
    "management":   subnets[100],  # fd3a:b2c1:d4e5:64::/64
}

for vlan, net in vlans.items():
    print(f"{vlan}: {net}")
```

## ULA vs Link-Local vs Global Unicast

| Property | ULA (fd::/8) | Link-Local (fe80::/10) | Global Unicast |
|---|---|---|---|
| Scope | Organization | Single link | Global |
| Routable | Within org | Never | Globally |
| Requires prefix delegation | No | No | Yes (ISP) |
| Stable (no renumbering) | Yes | Yes | No (ISP changes) |
| DNS registration | Internal only | No | Public |

## Filtering ULA at Network Boundaries

```bash
# Never route ULA to the internet
ip6tables -A FORWARD -s fc00::/7 -o eth-external -j DROP
ip6tables -A FORWARD -d fc00::/7 -i eth-external -j DROP

# BIRD/FRR: filter ULA from external BGP peers
# In route policy:
# if dest ~ [ fc00::/7+ ] then reject;
```

## Conclusion

ULA (`fd00::/8` specifically) provides stable, private IPv6 addressing for internal networks. Generate a random 40-bit Global ID per organization and use it consistently across all sites. Unlike IPv4 private space, ULA uniqueness (via random Global ID) makes VPN and network merges simpler. Monitor ULA subnet reachability within your organization using OneUptime internal health checks.
