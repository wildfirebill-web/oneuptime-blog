# How to Understand the 6to4 Address Space (2002::/16)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6to4, Tunneling, RFC 3056, Transition, Networking

Description: Understand the 6to4 address space 2002::/16 (RFC 3056) that encodes an IPv4 address into an IPv6 prefix for automatic tunneling over IPv4 infrastructure.

## Introduction

6to4 (RFC 3056) is a transition mechanism that automatically tunnels IPv6 over IPv4 networks. The `2002::/16` prefix encodes a host's public IPv4 address in the address, enabling IPv6 connectivity without ISP support.

## 6to4 Address Format

```
2002:AABB:CCDD::/48
  2002    = 6to4 prefix
  AABB    = first 16 bits of IPv4 address (hex)
  CCDD    = last 16 bits of IPv4 address (hex)

Example:
  IPv4: 192.0.2.1
  6to4: 2002:c000:0201::/48
  (192 = 0xC0, 0 = 0x00, 2 = 0x02, 1 = 0x01)
```

## Computing a 6to4 Address

```python
import ipaddress

def ipv4_to_6to4_prefix(ipv4: str) -> str:
    """Convert an IPv4 address to its 6to4 prefix."""
    v4 = ipaddress.IPv4Address(ipv4)
    v4_int = int(v4)

    # 6to4 = 2002::/16 + IPv4 (32 bits) + zeros
    prefix_int = 0x20020000000000000000000000000000
    ipv4_embedded = v4_int << 80  # Position IPv4 at bits 16-47
    addr_int = (0x2002 << 112) | (v4_int << 80)

    return str(ipaddress.IPv6Address(addr_int)) + "/48"

print(ipv4_to_6to4_prefix("192.0.2.1"))  # 2002:c000:201::/48

def is_6to4(addr: str) -> bool:
    a = ipaddress.IPv6Address(addr)
    return a in ipaddress.IPv6Network("2002::/16")
```

## Why 6to4 Is Deprecated

6to4 has significant operational problems:
- Relies on anycast `192.88.99.0/24` relays (anycast address deprecated in RFC 7526)
- Asymmetric routing causes connectivity failures
- Security issues from anonymous relay operators

```bash
# Disable 6to4 on Linux
# Remove any 6to4 tunnel interfaces
ip tunnel del tun6to4 2>/dev/null

# Block 6to4 at network boundaries
ip6tables -A FORWARD -s 2002::/16 -j DROP
ip6tables -A FORWARD -d 2002::/16 -j DROP

# Check for active 6to4 interfaces
ip -6 tunnel show | grep "6to4\|sit"
```

## Conclusion

The 6to4 address space `2002::/16` is deprecated and should not be used in new deployments. Replace any 6to4 tunnels with native IPv6, 464XLAT, or DS-Lite. Filter `2002::/16` at borders and monitor with OneUptime for unexpected usage.
