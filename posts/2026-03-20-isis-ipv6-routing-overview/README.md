# How to Understand IS-IS for IPv6 Routing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, Routing, Link-State, Networking

Description: Understand how IS-IS (Intermediate System to Intermediate System) protocol supports IPv6 routing through TLV extensions and multi-topology support.

## Overview

IS-IS is a link-state routing protocol originally designed for ISO/OSI networking but extended to support IP routing. Unlike OSPFv3, IS-IS is not a separate protocol for IPv6 — instead, IPv6 support was added to existing IS-IS via new TLV (Type-Length-Value) extensions.

## IS-IS Architecture for IPv6

```mermaid
graph TD
    IS_IS[IS-IS Protocol] --> TLV232[TLV 232 - IPv6 Interface Address]
    IS_IS --> TLV236[TLV 236 - IPv6 Reachability]
    IS_IS --> MT[Multi-Topology RFC 5120]
    MT --> MT2[MT-ID 2 - IPv6 Unicast]
    MT --> MT3[MT-ID 3 - IPv6 Multicast]
```

## How IS-IS Handles IPv6

IS-IS uses TLV extensions to carry IPv6 information in its Link State PDUs (LSPs):

| TLV | Number | Content |
|-----|--------|---------|
| IPv6 Interface Addresses | 232 | Link-local and global addresses on each interface |
| IPv6 Reachability | 236 | IPv6 prefixes (like Type 1 LSA in OSPF) |
| Multi-Topology Reachable IS | 222 | Adjacency for a specific topology |
| Multi-Topology IS Neighbor | 223 | Multi-topology neighbor information |

## IS-IS vs OSPFv3 for IPv6

| Feature | IS-IS | OSPFv3 |
|---------|-------|--------|
| Protocol family | ISO/CLNS-based | TCP/IP based |
| Transport | Runs directly over Layer 2 | UDP/IPv6 |
| IPv6 support | TLV extensions to existing IS-IS | New protocol (RFC 5340) |
| Multi-topology | RFC 5120 | RFC 5838 |
| Deployment | ISP/service provider heavy | Enterprise and SP |
| Authentication | MD5/SHA (in TLVs) | IPsec |

## IS-IS Levels

IS-IS uses two levels of hierarchy:
- **Level 1**: Intra-area routing (like OSPF intra-area)
- **Level 2**: Inter-area routing (like OSPF backbone)
- **L1/L2**: Router is part of both levels (like an OSPF ABR)

## Multi-Topology IS-IS (MT-ISIS)

Multi-Topology IS-IS (RFC 5120) allows separate topologies for IPv4 and IPv6 within the same IS-IS process. MT-ID 0 = Standard (IPv4), MT-ID 2 = IPv6 Unicast:

```mermaid
graph LR
    IS_IS_Process --> MT_IPv4[MT-ID 0 - IPv4 Unicast]
    IS_IS_Process --> MT_IPv6[MT-ID 2 - IPv6 Unicast]
    MT_IPv4 --> RIB_IPv4[IPv4 Routing Table]
    MT_IPv6 --> RIB_IPv6[IPv6 Routing Table]
```

## IS-IS Adjacency for IPv6

IS-IS adjacencies run directly over Layer 2 (Ethernet, HDLC) — not over IPv6. This is a key distinction from OSPFv3. An IS-IS adjacency can form even if IPv6 is not configured, and then carry IPv6 routes via TLVs.

## When to Choose IS-IS for IPv6

IS-IS is preferred when:
- You are building a service provider or large-scale ISP network
- You already run IS-IS for IPv4 and want to add IPv6 (easy extension)
- You need multi-topology support with separate IPv4/IPv6 path computation

## Summary

IS-IS supports IPv6 through TLV 232 (IPv6 interface addresses) and TLV 236 (IPv6 reachability). Unlike OSPFv3, IS-IS runs directly over Layer 2 — not over IP. Multi-Topology IS-IS (RFC 5120) enables independent IPv4 and IPv6 forwarding topologies. IS-IS is the dominant routing protocol in large ISP backbones and is commonly deployed in large-scale data center fabrics.
