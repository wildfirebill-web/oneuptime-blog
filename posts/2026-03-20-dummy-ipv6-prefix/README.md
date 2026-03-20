# How to Understand the Dummy IPv6 Prefix (100:0:0:1::/64)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Dummy Prefix, 100:0:0:1::/64, RFC 9003, Routing, Null Route

Description: Understand the IPv6 Dummy Prefix 100:0:0:1::/64 (RFC 9003), its use for dummy routing purposes, and how it differs from the Discard-Only block 100::/64.

## Introduction

`100:0:0:1::/64` is a specific /64 within the `100::/64` discard-only address block. RFC 9003 designates it specifically for use as a dummy prefix in scenarios where a routing protocol requires a valid prefix to advertise but the operator wants traffic discarded. This avoids using documentation or loopback addresses as routing placeholders.

## Relationship to the Discard-Only Block

```text
100::/64 - Discard-Only Address Block (RFC 6666)
  Used for RTBH (Remote Triggered Black Hole) filtering
  All traffic to 100::/64 should be discarded

100:0:0:1::/64 - Dummy Prefix (RFC 9003)
  A specific /64 within the discard-only space
  Used as a placeholder/dummy in routing protocols
  Traffic discarded just as with the parent block
```

## Use Cases for Dummy Prefixes

```bash
# 1. BGP route reflector - needs a valid prefix to stay "active"

# A BGP session that only redistributes other routes
# needs at least one locally originated prefix to stay UP
ip -6 route add blackhole 100:0:0:1::/64

# FRR: advertise dummy prefix via BGP
router bgp 65001
  address-family ipv6 unicast
    network 100:0:0:1::/64
  !
!

# 2. Placeholder for testing routing protocol configuration
# When you want to test BGP/OSPF advertisement without real traffic
ip -6 route add 100:0:0:1::/64 dev null

# 3. Route reflector confederation - aggregate placeholder
# Advertise a summary route that points to discard
ip -6 route add blackhole 100:0:0:1::/64
```

## Python: Identifying Dummy vs Discard Prefixes

```python
import ipaddress

DISCARD_BLOCK = ipaddress.IPv6Network("100::/64")
DUMMY_PREFIX = ipaddress.IPv6Network("100:0:0:1::/64")

def classify_100_prefix(addr_str: str) -> str:
    """Classify addresses in the 100::/64 block."""
    try:
        # Try as network
        net = ipaddress.IPv6Network(addr_str, strict=False)
        if net == DUMMY_PREFIX:
            return "Dummy Prefix (RFC 9003)"
        if net.subnet_of(DISCARD_BLOCK):
            return "Discard-Only (RFC 6666)"
    except ValueError:
        pass

    try:
        addr = ipaddress.IPv6Address(addr_str)
        if addr in DUMMY_PREFIX:
            return "Dummy Prefix (RFC 9003)"
        if addr in DISCARD_BLOCK:
            return "Discard-Only (RFC 6666)"
    except ValueError:
        pass

    return "Not in 100::/64"

# Tests
print(classify_100_prefix("100:0:0:1::/64"))   # Dummy Prefix
print(classify_100_prefix("100:0:0:1::1"))     # Dummy Prefix
print(classify_100_prefix("100::1"))            # Discard-Only
print(classify_100_prefix("100::"))             # Discard-Only
print(classify_100_prefix("100:0:0:2::1"))      # Discard-Only (different /64)
```

## Routing Configuration Examples

```bash
# Cisco IOS-XR: null route for dummy prefix
ipv6 route 100:0:0:1::/64 Null0 description "Dummy prefix per RFC 9003"

# Juniper Junos
set routing-options rib inet6.0 static route 100:0:0:1::/64 discard

# FRR
ipv6 route 100:0:0:1::/64 Null0

# Linux
ip -6 route add blackhole 100:0:0:1::/64
```

## Conclusion

The dummy IPv6 prefix `100:0:0:1::/64` is a designated placeholder within the discard-only space for routing protocol use. It allows operators to satisfy the need for a valid, routable prefix in routing configurations without using real addresses. Understand that all traffic to this prefix will be discarded. Monitor routing protocol sessions that use this dummy prefix with OneUptime to ensure they remain healthy.
