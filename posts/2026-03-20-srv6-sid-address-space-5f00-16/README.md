# How to Understand the SRv6 SID Address Space (5f00::/16)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, 5f00, SID, RFC 9602, IPv6 Addressing, Networking

Description: Understand the IANA-allocated 5f00::/16 address block for SRv6 Segment Identifiers, its structure, and how it simplifies SRv6 deployment and filtering.

## Introduction

RFC 9602 allocates `5f00::/16` as the global SRv6 SID address space. This provides a well-known, universally recognizable prefix for SRv6 deployments, enabling consistent hardware optimization and simpler access control compared to operator-specific prefixes.

## Why 5f00::/16?

Before RFC 9602, operators used their own allocated prefixes for SIDs, making it impossible for network equipment to optimize specifically for SRv6 SIDs. The dedicated `5f00::/16` allocation allows:

1. Hardware vendors to optimize forwarding for this prefix
2. Operators to consistently filter SRv6 SIDs at network boundaries
3. Monitoring systems to identify SRv6 traffic universally

## Address Space Properties

```python
import ipaddress

# 5f00::/16 properties
block = ipaddress.IPv6Network("5f00::/16")

print(f"Network address: {block.network_address}")  # 5f00::
print(f"Broadcast: {block.broadcast_address}")       # 5fff:ffff:...
print(f"Total addresses: {block.num_addresses}")     # 2^112

# Globally routable: Yes
# Source valid: Yes
# Destination valid: Yes
# Forwardable: Yes

def is_srv6_sid(addr: str) -> bool:
    """Check if an IPv6 address is in the SRv6 SID space."""
    try:
        a = ipaddress.IPv6Address(addr)
        return a in ipaddress.IPv6Network("5f00::/16")
    except ValueError:
        return False

print(is_srv6_sid("5f00:1:1::1"))  # True
print(is_srv6_sid("5fff::"))       # True
print(is_srv6_sid("6000::"))       # False
```

## Allocating from 5f00::/16

```
Recommended hierarchy:
  5f00:SITE:NODE::/48   — Node locator
  5f00:SITE:NODE:FUNC:: — Specific SID function

Example for a 3-site network:
  Site 1: 5f00:0001::/32
    R1:    5f00:0001:0001::/48
    R2:    5f00:0001:0002::/48
  Site 2: 5f00:0002::/32
  Site 3: 5f00:0003::/32
```

## Filtering Configuration

```bash
# Allow SRv6 SID traffic within your AS
ip6tables -A FORWARD -d 5f00::/16 \
  -s 5f00::/16 -j ACCEPT  # SRv6 internal

# Block SRv6 SIDs from external untrusted sources
ip6tables -A FORWARD -d 5f00::/16 \
  -i eth-external -j DROP
```

## Conclusion

The `5f00::/16` SRv6 SID allocation provides a globally recognizable, hardware-optimizable prefix for SRv6 deployments. New SRv6 deployments should use this space. Use OneUptime to monitor locator prefix reachability within `5f00::/16` as a health indicator for your SRv6 infrastructure.
