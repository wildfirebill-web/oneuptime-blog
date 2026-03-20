# How to Understand IPv6 Route Types (Connected, Static, Dynamic)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Routing, Connected Routes, Static Routes, Dynamic Routing

Description: Learn the three main IPv6 route types - connected, static, and dynamic - how each is created, and when to use each type in network design.

## Overview

IPv6 routes come from three sources: they are automatically created when interfaces come up (connected), manually configured by an administrator (static), or learned from routing protocols (dynamic). Understanding each type helps you design resilient and manageable IPv6 networks.

## Connected Routes

Connected routes are automatically added by the kernel when an IPv6 address is assigned to an interface. They represent directly attached networks that require no gateway.

```bash
# Assign an address to create connected routes

sudo ip -6 addr add 2001:db8:1::1/64 dev eth0

# The kernel automatically adds:
ip -6 route show | grep "proto kernel"
# 2001:db8:1::/64 dev eth0 proto kernel scope link src 2001:db8:1::1 metric 256
# fe80::/64 dev eth0 proto kernel scope link src fe80::1 metric 256
```

Connected routes have `proto kernel` and `scope link` - meaning no gateway is needed, the destination is directly reachable.

## Static Routes

Static routes are manually configured by an administrator and remain until explicitly removed (or the system reboots if not persisted):

```bash
# Add a static route - appears as 'proto static'
sudo ip -6 route add 2001:db8:2::/48 via fe80::router dev eth0

ip -6 route show | grep "proto static"
# 2001:db8:2::/48 via fe80::router dev eth0 proto static metric 1024
```

Use cases for static routes:
- Small networks with stable topology
- Routes that must never change
- Stub networks with a single exit point
- Floating routes (high metric backup routes)

## Dynamic Routes

Dynamic routes are learned from routing protocols (OSPFv3, BGP, RIPng, IS-IS, EIGRP). They adapt automatically to topology changes:

```bash
# Routes learned by OSPFv3 via FRRouting
ip -6 route show | grep "proto ospf\|proto bgp\|proto ripng"
# 2001:db8:3::/48 via fe80::2 dev eth1 proto ospf metric 20
# 2001:db8:4::/48 via fe80::3 dev eth2 proto bgp metric 200
```

Dynamic routing protocols install routes with their own protocol identifier (`proto ospf`, `proto bgp`, etc.).

## Route Protocol Numbers

```bash
# View protocol number to name mappings
cat /etc/iproute2/rt_protos

# Common IPv6 protocol values:
# 0    = unspecified
# 2    = kernel (connected routes)
# 4    = static
# 11   = ospf3
# 186  = bgp
# 189  = isis
```

## Administrative Distance (Preference)

When multiple protocols offer routes to the same prefix, preference is determined by **administrative distance** (in Cisco terminology) or **metric**:

| Route Source | Linux Default Metric | Cisco AD |
|-------------|---------------------|----------|
| Connected | 256 | 0 |
| Static | 1024 | 1 |
| OSPFv3 | Varies | 110 |
| BGP | Varies | 20 (eBGP) / 200 (iBGP) |
| RIPng | Varies | 120 |

## Floating Static Routes (Backup)

A static route with a high metric acts as a backup to a dynamic route:

```bash
# Main route via OSPF (metric 20)
# Backup static route with higher metric (only used if OSPF route disappears)
sudo ip -6 route add 2001:db8:5::/48 via fe80::backup dev eth2 metric 2000
```

If the OSPF route is removed, the static backup at metric 2000 becomes active.

## Viewing Route Sources Together

```bash
# Show all routes with their protocol sources
ip -6 route show
# ::1 dev lo proto kernel scope host metric 256
# 2001:db8:1::/64 dev eth0 proto kernel scope link src 2001:db8::1  (connected)
# 2001:db8:2::/48 via fe80::router dev eth0 proto static             (static)
# 2001:db8:3::/48 via fe80::2 dev eth1 proto ospf metric 20         (dynamic)
# ::/0 via fe80::1 dev eth0 proto ra metric 1024                     (RA-learned default)
```

## Summary

IPv6 has three route types: **connected** (proto kernel, created by assigning addresses), **static** (proto static, manually configured), and **dynamic** (proto ospf/bgp/ripng, learned from routing protocols). Each serves a different purpose. In practice, most networks use all three together - connected for local subnets, static for simple stub sites, and dynamic protocols for scalable multi-router topologies.
