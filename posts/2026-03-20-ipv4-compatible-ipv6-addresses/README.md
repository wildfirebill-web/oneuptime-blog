# How to Understand IPv4-Compatible IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPv4, Transition Mechanisms, RFC 4291, Networking

Description: Understand IPv4-compatible IPv6 addresses, why they were deprecated, how they differ from IPv4-mapped addresses, and what transition mechanisms replaced them.

## Introduction

IPv4-compatible IPv6 addresses (`::w.x.y.z` or `::0:w.x.y.z/96`) were an early IPv6 transition mechanism defined in RFC 1884 (later updated by RFC 2373). They embedded an IPv4 address in the lower 32 bits of an IPv6 address with all other bits set to zero. While now deprecated by RFC 4291, they appear in legacy documentation and are important to understand for historical context.

## Address Structure

```
|<-------------- 96 bits of zeros ------------->|<-- 32 bits -->|
|  0000:0000:0000:0000:0000:0000                 |  IPv4 address |

Example:
IPv4:  192.168.1.1
IPv4-compatible IPv6: ::192.168.1.1  (or equivalently ::c0a8:0101)

Full 128-bit representation:
0000:0000:0000:0000:0000:0000:c0a8:0101
```

## IPv4-Compatible vs IPv4-Mapped

These two address types are easily confused:

| Property | IPv4-Compatible | IPv4-Mapped |
|---|---|---|
| Prefix | `::w.x.y.z` (all zeros) | `::ffff:w.x.y.z` |
| 80-bit prefix | All zeros | All zeros |
| Bits 81-96 | `0x0000` | `0xFFFF` |
| Status | **Deprecated** (RFC 4291) | Active, widely used |
| Use case | Obsolete automatic tunneling | Dual-stack sockets |

```python
import ipaddress

# IPv4-compatible (DEPRECATED - all zeros in bits 81-96)
compat = ipaddress.IPv6Address("::192.168.1.1")
print(f"IPv4-compatible: {compat}")
print(f"Is v4-compatible: {compat.ipv4_mapped}")  # None in Python 3.x
# Python 3.9+ treats ::w.x.y.z as a regular IPv6 address

# IPv4-mapped (still used)
mapped = ipaddress.IPv6Address("::ffff:192.168.1.1")
print(f"IPv4-mapped: {mapped}")
print(f"IPv4 address: {mapped.ipv4_mapped}")  # 192.168.1.1
```

## Why IPv4-Compatible Addresses Were Deprecated

IPv4-compatible addresses were designed for an automatic tunneling mechanism called **6over4** (RFC 2529), where IPv4 hosts with no explicit tunnel configuration could automatically reach IPv6 destinations by embedding their IPv4 address in an IPv6 address.

**Problems that led to deprecation:**
1. Required every IPv4 host to also run IPv6, which never happened at scale
2. Created security issues — any IPv4 host could send IPv6 packets to any other IPv4 host
3. The all-zeros prefix collided with the IPv6 unspecified address (`::`) and loopback (`::1`)
4. More robust transition mechanisms (6to4, Teredo, NAT64) rendered it obsolete

## Modern Replacement Mechanisms

What replaced IPv4-compatible addresses:

**6to4 (RFC 3056)**: Uses the `2002::/16` prefix with the IPv4 address embedded in bits 17-48:
```
IPv4: 198.51.100.1 → 6to4: 2002:c633:6401::/48
```

**ISATAP (RFC 5214)**: Intra-site tunneling using `::0:5efe:w.x.y.z` IIDs.

**NAT64 + DNS64 (RFC 6146)**: Translates between IPv6-only clients and IPv4-only servers, the preferred modern approach.

## Identifying Legacy IPv4-Compatible Addresses

```python
# Python: detect an IPv4-compatible address
def is_ipv4_compatible(addr_str):
    """
    Returns True if addr is an IPv4-compatible IPv6 address.
    These have the format ::w.x.y.z with all-zero prefix (no 0xFFFF).
    """
    try:
        addr = ipaddress.IPv6Address(addr_str)
        addr_int = int(addr)
        # Check: upper 96 bits must be all zeros
        # AND it must not be ::1 (loopback) or :: (unspecified)
        upper_96 = addr_int >> 32
        lower_32 = addr_int & 0xFFFFFFFF
        if upper_96 == 0 and lower_32 != 0 and lower_32 != 1:
            return True
        return False
    except ValueError:
        return False

print(is_ipv4_compatible("::192.168.1.1"))   # True (deprecated)
print(is_ipv4_compatible("::ffff:192.168.1.1"))  # False (this is mapped)
print(is_ipv4_compatible("::1"))              # False (loopback)
```

## Legacy System Considerations

If you encounter IPv4-compatible addresses in legacy configurations:

```bash
# Old Cisco IOS syntax for 6to4 (not IPv4-compatible)
# interface Tunnel0
#  ipv6 address 2002:c633:6401::1/128
#  tunnel mode ipv6ip 6to4

# Wireshark filter to catch IPv4-compatible packets (rare)
# ip.version == 6 and ipv6.addr matches "^::[0-9]"

# Check if your system generates IPv4-compatible addresses
ip -6 addr show | grep "^0:0:0:0:0:0:"
```

## Conclusion

IPv4-compatible IPv6 addresses are a deprecated relic of early IPv6 transition planning. While unlikely to be encountered in modern deployments, understanding them prevents confusion when reading older RFCs and network documentation. Today, IPv4-mapped addresses (`::ffff:/96`) serve the dual-stack socket role, and NAT64/DNS64 handles IPv6-only to IPv4-only communication without embedding IPv4 in IPv6 addresses.
