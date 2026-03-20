# How to Understand Differences Between RIPv2 and RIPng

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RIPng, RIPv2, IPv6, Routing, Comparison

Description: A detailed comparison of RIPv2 and RIPng covering protocol changes, authentication differences, and operational considerations for IPv6 migration.

## Overview

RIPng (RFC 2080) is not a direct IPv6 port of RIPv2 - it includes architectural changes to align with IPv6 principles. Understanding these differences helps engineers migrating from IPv4 to IPv6 routing.

## Detailed Comparison Table

| Feature | RIPv2 | RIPng |
|---------|-------|-------|
| RFC | 2453 | 2080 |
| Protocol | UDP/IPv4 | UDP/IPv6 |
| Port | 520 | 521 |
| Multicast address | 224.0.0.9 | ff02::9 (link-local scope) |
| Broadcast | 255.255.255.255 (old) | Not used |
| Authentication | Built-in (plain/MD5 in RTE) | IPsec (external) |
| Next-hop encoding | In each RTE | Special RTE type |
| Max hop count | 15 | 15 |
| Packet size | 512 bytes | Up to IPv6 MTU |
| Address size | 32-bit | 128-bit |

## Authentication: The Key Difference

RIPv2 has a built-in authentication RTE (Route Table Entry) using plain text or MD5. RIPng removes this entirely and relies on IPsec:

```text
! RIPv2 MD5 Authentication (Cisco)
interface GigabitEthernet0/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_KEY

! RIPng Authentication - use IPsec (FRRouting)
interface eth0
 ipv6 rip RIPNG enable
! IPsec security policy must be configured separately
```

## Multicast Scope Change

RIPv2 uses `224.0.0.9` (IPv4 multicast, link-local scope). RIPng uses `ff02::9` which is an IPv6 link-local multicast address - packets never leave the local link:

```bash
# Verify RIPng is joined to ff02::9

ip -6 maddr show dev eth0 | grep "ff02::9"
```

## Next-Hop Encoding Difference

In RIPv2, each RTE contains a 4-byte next-hop field. In RIPng, the next-hop for a group of routes is specified once using a special "next-hop RTE" (with metric 0xFF) followed by the routes that use it:

```text
RIPv2: Each RTE = [prefix, netmask, next-hop, metric]
RIPng: NextHop RTE (metric=0xFF) + RTEs = [prefix, prefix-len, metric]
```

## Packet Size Differences

RIPv2 is limited to 512 bytes per packet (20 bytes IP + 8 bytes UDP + 484 bytes payload = 25 RTEs). RIPng can use the full IPv6 MTU (typically 1500 bytes), allowing more routes per packet.

## Running Both Simultaneously

On dual-stack routers, both RIPv2 and RIPng can run independently:

```bash
# FRRouting - both protocols active simultaneously
# /etc/frr/daemons
ripd=yes
ripngd=yes

# In vtysh:
router rip             ! RIPv2 for IPv4
router ripng           ! RIPng for IPv6
```

```text
! Cisco - both on the same interface
interface GigabitEthernet0/0
 ip rip send version 2    ! RIPv2
 ipv6 rip RIPNG enable    ! RIPng
```

## When to Choose RIPng

Choose RIPng over RIPv2 when:
- You are deploying a new IPv6-only network
- The network is small (under 15 hops)
- Migration from an IPv4 RIP network to IPv6

Avoid RIPng for:
- Large networks (use OSPFv3 or BGP)
- High-availability requirements (no fast convergence)
- Networks needing complex traffic engineering

## Summary

RIPng adapts RIPv2 for IPv6 with protocol number and address changes, and moves authentication to IPsec. The 15-hop limit is unchanged. RIPng uses UDP port 521 and ff02::9 multicast. On dual-stack networks, RIPv2 and RIPng run as completely independent processes and can coexist on the same interfaces.
