# How to Calculate IPv4 Address Requirements for a New Office

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Network Planning, Subnetting, Office Networks, DHCP

Description: Calculate how many IPv4 addresses a new office needs by inventorying device types, applying growth and DHCP overhead factors, and selecting appropriately sized subnets.

## Introduction

Allocating too small a subnet for an office causes address exhaustion; too large wastes the address space and complicates security. A structured calculation ensures you pick the right prefix length from the start.

## Device Inventory Worksheet

```
Device Category         Count   Notes
─────────────────────────────────────────────────
Workstations (wired)     50     DHCP
Workstations (wireless)  80     DHCP
Laptops (wireless)       60     DHCP
VoIP phones              50     Separate VLAN
Printers / MFPs          10     Static
Servers / NAS             5     Static
Network devices           8     Static (switches, APs, router)
Guest Wi-Fi             100     Separate VLAN, dynamic
IoT devices              20     Separate VLAN
─────────────────────────────────────────────────
Totals per VLAN:
  Corporate wired+wireless: 190 devices
  VoIP:                      50
  Guest:                    100
  IoT:                       20
  Infrastructure:            13
```

## Subnet Sizing Formula

```
Required addresses = devices × 1.3 (30% growth buffer)
                     + DHCP pool reserve (10-15% of pool)

Round up to next power-of-2 subnet.
```

## Python Calculator

```python
import ipaddress
import math

def required_prefix(device_count: int, growth_pct: float = 0.30) -> int:
    """Return the smallest CIDR prefix that fits the device count."""
    needed = int(device_count * (1 + growth_pct)) + 2  # +2 for network/bcast
    bits = math.ceil(math.log2(needed))
    prefix = 32 - bits
    return max(prefix, 1)   # clamp

segments = {
    "Corporate":      190,
    "VoIP":            50,
    "Guest":          100,
    "IoT":             20,
    "Infrastructure":  13,
}

base = ipaddress.IPv4Network("10.50.0.0/20")
subnets = list(base.subnets(new_prefix=24))  # pre-allocate /24s

for i, (name, count) in enumerate(segments.items()):
    prefix = required_prefix(count)
    needed_hosts = int(count * 1.3)
    print(f"{name:<18} devices={count:3d}  "
          f"recommended=/{prefix}  "
          f"({2**(32-prefix)-2} usable)  "
          f"growth headroom={2**(32-prefix)-2-needed_hosts}")
```

## Sample Output and Subnet Selection

```
Corporate         devices=190  recommended=/24  (254 usable)  headroom=7
VoIP              devices= 50  recommended=/26  ( 62 usable)  headroom=  7
Guest             devices=100  recommended=/25  (126 usable)  headroom= 2
IoT               devices= 20  recommended=/27  ( 30 usable)  headroom= 4
Infrastructure    devices= 13  recommended=/28  ( 14 usable)  headroom= 1 → use /27
```

## DHCP Scope Sizing

```
For a /24 (254 usable):
  Static reservations:  10  (servers, printers)
  DHCP pool:           220  (lease entries)
  Admin/growth reserve: 24
  ─────────────────────────
  Total:               254
```

## Recommended Addressing for a 200-Person Office

```
10.50.0.0/22   — Office total allocation
  10.50.0.0/24   VLAN 10  Corporate
  10.50.1.0/25   VLAN 20  VoIP
  10.50.1.128/25 VLAN 30  Guest
  10.50.2.0/27   VLAN 40  IoT
  10.50.2.32/27  VLAN 99  Infrastructure
```

## Conclusion

Start with a device inventory, apply a 30% growth buffer, add DHCP overhead, and choose the next power-of-two subnet. Segment VoIP, guest, IoT, and corporate traffic into separate VLANs. Allocate a parent block large enough to summarize all office subnets into a single route advertisement.
