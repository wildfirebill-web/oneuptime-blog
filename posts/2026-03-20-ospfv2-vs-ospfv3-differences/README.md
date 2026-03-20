# How to Understand Differences Between OSPFv2 and OSPFv3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPFv3, OSPFv2, IPv6, Routing, Comparison

Description: A detailed comparison of OSPFv2 and OSPFv3 covering protocol changes, LSA types, authentication, and address family support.

## Overview

OSPFv3 (RFC 5340) is not simply OSPFv2 with IPv6 addresses substituted — it underwent significant architectural changes. Understanding these differences is essential for network engineers transitioning from IPv4 OSPF to IPv6 environments.

## Side-by-Side Comparison

| Feature | OSPFv2 | OSPFv3 |
|---------|--------|--------|
| RFC | RFC 2328 | RFC 5340 |
| IP Version | IPv4 | IPv6 |
| Neighbor formation | IPv4 addresses | IPv6 link-local (fe80::) |
| Authentication | Built-in (plain, MD5, SHA) | IPsec AH/ESP (RFC 4552) |
| Router ID | Highest IPv4 addr or loopback | Must be configured manually if no IPv4 |
| Addressing in LSAs | Included in LSAs | Separated into new LSA types |
| Multiple instances per link | No | Yes (Instance ID field) |
| Address families | IPv4 only | IPv6 (base), IPv4+IPv6 (RFC 5838) |

## LSA Type Differences

OSPFv3 introduces new LSA types and removes addressing from existing ones:

| LSA Type | OSPFv2 Name | OSPFv3 Name | Change |
|----------|-------------|-------------|--------|
| 1 | Router LSA | Router LSA | No addresses; uses link IDs |
| 2 | Network LSA | Network LSA | No addresses |
| 3 | Summary LSA | Inter-Area Prefix LSA | Carries prefixes instead of subnets |
| 4 | ASBR Summary | Inter-Area Router LSA | Changed content |
| 5 | AS External | AS External LSA | Carries IPv6 prefixes |
| 8 | N/A | Link LSA | NEW — carries link-local address + on-link prefixes |
| 9 | Opaque | Intra-Area Prefix LSA | NEW — carries intra-area prefixes |

## Authentication Changes

OSPFv2 has authentication built into its packet header. OSPFv3 removes this entirely and relies on IPsec:

```
! OSPFv2 MD5 authentication (Cisco)
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 secretkey

! OSPFv3 IPsec authentication (Cisco)
interface GigabitEthernet0/0
 ospfv3 authentication ipsec spi 256 sha1 <key-string>
```

## Running OSPFv2 and OSPFv3 Simultaneously

On dual-stack routers, both protocols run independently:

```
! Cisco: Enable both on the same interface
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 ipv6 address 2001:db8::1/64
 ip ospf 1 area 0         ! OSPFv2
 ospfv3 1 ipv6 area 0     ! OSPFv3
```

```
# FRRouting: Both protocols active
router ospf
 network 10.0.0.0/24 area 0

router ospf6
 interface eth0 area 0.0.0.0
```

## Key Behavioral Differences

**Neighbor Formation**: OSPFv2 uses IPv4 addresses; OSPFv3 uses fe80:: addresses. This means OSPFv3 adjacencies survive IPv6 global address renumbering.

**Multiple Instances**: OSPFv3 supports multiple independent instances on the same link via the Instance ID field (0-255). OSPFv2 has no equivalent.

**Address Separation**: OSPFv3 cleanly separates topology (Router and Network LSAs) from addressing (Intra-Area Prefix LSAs). This architectural improvement makes OSPFv3 more suitable for RFC 5838 multi-topology extensions.

## Summary

OSPFv3 is architecturally cleaner than OSPFv2 — it separates topology from addressing, uses IPsec for authentication, and supports multiple instances per link. For engineers familiar with OSPFv2, the concepts (areas, DR/BDR, LSA flooding) are identical, but the implementation details differ significantly, especially around authentication and LSA types.
