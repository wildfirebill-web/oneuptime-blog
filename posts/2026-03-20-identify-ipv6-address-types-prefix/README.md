# How to Identify IPv6 Address Types from Their Prefix

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Networking, Address Types, Prefixes, Unicast, Multicast, Anycast

Description: Identify all major IPv6 address types - global unicast, link-local, multicast, loopback, unspecified, and unique local - by their prefix ranges.

## Introduction

IPv6 address types are distinguished by their prefix bits. Unlike IPv4 where class A/B/C divisions are historical, IPv6 prefix-based classification is authoritative and consistent across all implementations.

## IPv6 Address Type Reference Table

| Type | Prefix | Example |
|------|--------|---------|
| Loopback | ::1/128 | ::1 |
| Unspecified | ::/128 | :: |
| Global Unicast | 2000::/3 | 2001:db8::1 |
| Link-Local Unicast | fe80::/10 | fe80::1 |
| Unique Local Unicast | fc00::/7 | fd00:1:2:3::1 |
| Multicast | ff00::/8 | ff02::1 |
| IPv4-Mapped | ::ffff:0:0/96 | ::ffff:192.168.1.1 |
| IPv4-Compatible | ::/96 | ::192.168.1.1 (deprecated) |
| 6to4 | 2002::/16 | 2002:c0a8:0101:: (deprecated) |
| Teredo | 2001::/32 | 2001::... |
| Documentation | 2001:db8::/32 | 2001:db8::1 |

## Python Type Detector

```python
import ipaddress

def identify_ipv6_type(addr: str) -> str:
    try:
        ip = ipaddress.IPv6Address(addr.split('%')[0])  # strip zone ID
    except ValueError:
        return "Invalid"

    if ip.is_loopback:
        return "Loopback (::1)"
    if ip.is_unspecified:
        return "Unspecified (::)"
    if ip.is_link_local:
        return "Link-Local (fe80::/10)"
    if ip.is_multicast:
        return f"Multicast (scope={multicast_scope(ip)})"
    if ip.is_private:
        return "Unique Local Unicast (fc00::/7)"
    if ip.ipv4_mapped:
        return f"IPv4-Mapped (::ffff:{ip.ipv4_mapped})"
    if ip in ipaddress.IPv6Network("2001:db8::/32"):
        return "Documentation (2001:db8::/32)"
    if ip in ipaddress.IPv6Network("2002::/16"):
        return "6to4 (2002::/16)"
    if ip in ipaddress.IPv6Network("2001::/32"):
        return "Teredo (2001::/32)"
    if ip in ipaddress.IPv6Network("2000::/3"):
        return "Global Unicast (2000::/3)"
    return "Reserved/Unknown"

def multicast_scope(ip: ipaddress.IPv6Address) -> str:
    scopes = {
        0x1: "interface-local",
        0x2: "link-local",
        0x4: "admin-local",
        0x5: "site-local",
        0x8: "organization-local",
        0xe: "global",
    }
    scope_nibble = int(str(ip).split(':')[0], 16) & 0xf
    return scopes.get(scope_nibble, f"unknown({scope_nibble:#x})")

tests = [
    "::1", "::", "fe80::1", "ff02::1", "fd00::1",
    "2001:db8::1", "::ffff:192.168.1.1", "2001:4860:4860::8888",
]
for addr in tests:
    print(f"{addr:40s} {identify_ipv6_type(addr)}")
```

## Multicast Scope Breakdown

```text
ff01::  - Interface-local multicast (loopback scope)
ff02::  - Link-local multicast (on-link)
ff05::  - Site-local multicast
ff0e::  - Global multicast

Common multicast groups:
ff02::1   - All nodes (link-local)
ff02::2   - All routers (link-local)
ff02::fb  - mDNS
ff02::1:2 - DHCP agents
ff02::1:ff00:0/104 - Solicited-node multicast
```

## Unique Local vs Global Unicast

```text
Unique Local (fc00::/7):
  fc00::/8  - Not yet assigned
  fd00::/8  - Locally assigned (use this)
  Example:  fd12:3456:789a::1/48
  Scope:    Routing within organization only (not globally routable)

Global Unicast (2000::/3):
  Currently allocated:  2001::/16, 2600::/12, etc.
  Globally routable
  Example:  2600:1f18:1234:5678::1
```

## Conclusion

IPv6 address types are identified purely by prefix. `fe80::/10` is always link-local; `ff00::/8` is always multicast; `fc00::/7` (`fd00::/8` in practice) is unique-local (RFC 4193). Python's `ipaddress` module exposes `is_link_local`, `is_loopback`, `is_multicast`, and `is_private` properties for programmatic type detection.
