# How to Understand IPv6 Multicast vs IPv4 Multicast (IGMP)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, IGMP, MLD, Protocol Comparison

Description: A comparative analysis of IPv6 multicast (MLD) and IPv4 multicast (IGMP), highlighting the differences in protocol design, address ranges, and deployment considerations.

## Overview

Both IPv4 and IPv6 support multicast, but they use different protocols for group management:
- IPv4: **IGMP** (Internet Group Management Protocol) - IGMPv1, v2, v3
- IPv6: **MLD** (Multicast Listener Discovery) - MLDv1, v2

MLD is essentially IGMP redesigned as part of ICMPv6 with IPv6 address support.

## Protocol Comparison

| Feature | IPv4 IGMP | IPv6 MLD |
|---|---|---|
| RFC | RFC 3376 (IGMPv3) | RFC 3810 (MLDv2) |
| Part of | Standalone IP protocol | ICMPv6 (type 130-132, 143) |
| IP version | IPv4 only | IPv6 only |
| Message transport | IPv4 with Router Alert option | IPv6 with Hop-by-Hop Router Alert |
| Group address range | 224.0.0.0/4 | ff00::/8 |
| ASM range | 224.0.0.0/4 | ff0x::/16 |
| SSM range | 232.0.0.0/8 | ff3x::/32 |
| Link-local range | 224.0.0.0/24 | ff02::/16 |
| Source-specific (v3/v2) | IGMPv3 | MLDv2 |

## Address Range Differences

IPv4 multicast uses Class D addresses (224.0.0.0/4):
```text
224.0.0.0/24     - Link-local (routers don't forward)
224.0.1.0-238.255.255.255  - Global multicast
232.0.0.0/8      - SSM range
239.0.0.0/8      - Organization-local (similar to IPv6 site-local)
```

IPv6 multicast uses the ff00::/8 prefix with embedded scope:
```text
ff02::/16        - Link-local (routers don't forward)
ff05::/16        - Site-local
ff0e::/16        - Global multicast
ff3e::/32        - SSM (global scope)
```

## Key Protocol Differences

### 1. Protocol Encapsulation

**IGMP**: Carried directly in IPv4 as a separate protocol (protocol number 2):
```text
IPv4 Header (proto=2) | IGMP Message
```

**MLD**: Carried as ICMPv6 (protocol 58), with mandatory Hop-by-Hop Router Alert:
```text
IPv6 Header | Hop-by-Hop Options (Router Alert) | ICMPv6 (MLD Message)
```

The Hop-by-Hop extension header is mandatory for all MLD messages, ensuring routers on the path process the MLD packet even if they're not the IPv6 destination.

### 2. Link-Local Source Address Required

**IGMP**: Uses the interface's IPv4 address as source.

**MLD**: Must use a link-local IPv6 address as source (RFC 3590). This is why MLD works even before an interface gets a global IPv6 address - link-local addresses are always available.

```bash
# Verify MLD reports use link-local source

tcpdump -i eth0 -n 'icmp6 and ip6[40] == 143' | grep 'fe80'
# Source should always be a fe80:: address
```

### 3. Report Suppression

**IGMPv2**: Hosts suppress their reports when they hear another host already reporting for the same group.

**MLDv2**: No report suppression - each host reports independently. This is simpler but generates more traffic on busy links.

### 4. Scope Awareness

**IGMP**: Uses TTL to limit forwarding scope (local-scope groups use TTL=1).

**MLD**: Uses the scope field embedded in the multicast address (ff0**2**::/16 = link-local, ff0**5**::/16 = site-local). Routers make forwarding decisions based on the address itself, not packet headers.

## Configuring Both in a Dual-Stack Environment

For dual-stack networks, you need both IGMP and MLD:

```bash
# Verify both are running on Linux
# IGMP state
cat /proc/net/igmp6    # This is actually the IPv6 multicast socket table
cat /proc/net/igmp     # IPv4 IGMP table

# IPv4 multicast groups
ip maddr show
# IPv6 multicast groups
ip -6 maddr show
```

## Routing Protocol Comparison

| Routing Feature | IPv4 | IPv6 |
|---|---|---|
| Dense Mode | PIM-DM | PIM-DM (same RFC) |
| Sparse Mode | PIM-SM | PIM-SM (RFC 7761) |
| Source-Specific | PIM-SSM | PIM-SSM (same concept) |
| MSDP for RP sync | Yes | Not needed (Embedded RP) |
| Embedded RP | No | Yes (RFC 3956) |

## Summary

IPv6 MLD and IPv4 IGMP serve the same purpose (multicast group management) but have design differences: MLD is part of ICMPv6 (requires Hop-by-Hop extension header), uses link-local source addresses, supports scope via address embedding, and has no report suppression. IPv6's Embedded RP feature eliminates one of PIM-SM's biggest operational challenges. For new deployments, IPv6 multicast with MLDv2 and PIM-SSM is the recommended approach.
