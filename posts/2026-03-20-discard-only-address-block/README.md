# How to Understand the Discard-Only Address Block (100::/64)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DISCARD, Blackhole, 100::/64, RFC 6666, Networking

Description: Understand the Discard-Only Address Block 100::/64 (RFC 6666), its role as a remote-triggered blackhole target, and how to use it for traffic discarding.

## Introduction

`100::/64` is the IPv6 Discard-Only Address Block defined in RFC 6666. Traffic sent to any address in this range should be discarded. It is the IPv6 equivalent of IPv4's `192.0.2.0/24` discard prefix used in Remote Triggered Black Hole (RTBH) filtering. It is forwardable but not globally reachable.

## Key Properties

| Property | Value |
|---|---|
| Prefix | 100::/64 |
| RFC | RFC 6666 |
| Forwardable | Yes (used for RTBH) |
| Globally Reachable | No |
| Source | True (allowed, but unusual) |
| Destination | True (traffic discarded) |

## Remote Triggered Black Hole (RTBH) Filtering

```bash
# The typical RTBH use case:

# 1. Install a null route to 100::/64 on all routers
ip -6 route add 100::/64 dev null

# 2. When under DDoS, advertise the attacked prefix to 100::/64 via BGP
# Any traffic matching that route gets discarded at all routers

# Example: attacker targets 2001:db8::victim/128
# BGP RTBH: advertise 2001:db8::victim/128 → next-hop 100::1

# On all edge routers:
ip -6 route add blackhole 100::1/128
# BGP installs route to attacked IP → 100::1 → null
```

## Python: Detect Discard-Only Addresses

```python
import ipaddress

DISCARD_BLOCK = ipaddress.IPv6Network("100::/64")

def is_discard_only(addr_str: str) -> bool:
    """Return True if address is in the Discard-Only block."""
    try:
        addr = ipaddress.IPv6Address(addr_str)
        return addr in DISCARD_BLOCK
    except ValueError:
        return False

# Tests
print(is_discard_only("100::1"))        # True
print(is_discard_only("100::ffff"))     # True
print(is_discard_only("100::"))         # True
print(is_discard_only("100:0:0:1::"))   # False (different /64)
print(is_discard_only("::1"))           # False
```

## Router Configuration for RTBH

```bash
# Cisco IOS-XR: null route for RTBH
ipv6 route 100::/64 Null0

# Juniper Junos
set routing-options rib inet6.0 static route 100::/64 discard

# Linux
ip -6 route add blackhole 100::/64

# FRR
ipv6 route 100::/64 Null0

# BGP RTBH community (standard practice)
# When a prefix is BGP-advertised with community NO_EXPORT + 65535:666
# all receiving routers set next-hop to 100::1 and install null route
```

## Firewall Rules

```bash
# Block inbound traffic FROM 100::/64 (spoofed source)
ip6tables -A INPUT -s 100::/64 -j DROP

# Block outbound traffic TO 100::/64 (misconfiguration protection)
ip6tables -A OUTPUT -d 100::/64 -j DROP
```

## Distinguishing from Documentation and Loopback

```python
import ipaddress

special = {
    "100::/64": "Discard-Only (RFC 6666)",
    "100:0:0:1::/64": "NOT Discard-Only (out of range)",
    "::1/128": "Loopback",
    "2001:db8::/32": "Documentation",
}

for addr, expected in special.items():
    net = ipaddress.IPv6Network(addr, strict=False)
    in_discard = net.subnet_of(ipaddress.IPv6Network("100::/64")) if net.prefixlen >= 64 else False
    print(f"{addr}: discard={in_discard} ({expected})")
```

## Conclusion

`100::/64` is a dedicated IPv6 discard block primarily used for RTBH filtering. All routers should have a null route for this prefix. In BGP deployments, RTBH communities trigger null route installation network-wide to discard attack traffic. Monitor your BGP RTBH activations with OneUptime to track DDoS mitigation events.
