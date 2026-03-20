# How to Configure Multi-Topology IS-IS for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, Multi-Topology, MT-ISIS, Routing

Description: Learn how to configure Multi-Topology IS-IS (MT-ISIS) per RFC 5120 to enable separate IPv4 and IPv6 forwarding topologies within a single IS-IS process.

## Overview

Multi-Topology IS-IS (RFC 5120) allows a single IS-IS process to maintain separate link-state databases for different address families. This means IPv4 and IPv6 can have completely independent forwarding topologies, enabling gradual migration without requiring all routers to support both.

## Why Multi-Topology?

Without multi-topology, IPv6 prefixes are flooded in the same IS-IS topology as IPv4. This creates a problem: if a router has an IPv6 link down but IS-IS still shows it as reachable, IPv6 packets may be black-holed. MT-ISIS solves this by computing separate SPFs per topology.

## MT-ISIS Topology IDs

| MT-ID | Topology | RFC |
|-------|---------|-----|
| 0 | IPv4 Standard (default) | RFC 5120 |
| 2 | IPv6 Unicast | RFC 5120 |
| 3 | IPv6 Multicast | RFC 5120 |

## Configuring MT-ISIS on Cisco IOS

```text
! Enable IS-IS with multi-topology for IPv6
Router(config)# router isis
Router(config-router)# net 49.0001.0001.0001.0001.00    ! NSAP address
Router(config-router)# is-type level-2-only

! Enable multi-topology for IPv6
Router(config-router)# address-family ipv6
Router(config-router-af)# multi-topology    ! Enables MT-ID 2 for IPv6

! Enable IS-IS on the interface
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:db8:1::1/64
Router(config-if)# isis ipv6 metric 10    ! IPv6-specific interface metric
Router(config-if)# ip router isis         ! IPv4 IS-IS
Router(config-if)# ipv6 router isis       ! IPv6 IS-IS
```

## Configuring MT-ISIS on Juniper

```text
# JunOS MT-ISIS configuration

set protocols isis interface ge-0/0/0.0 level 2 metric 10
set protocols isis interface ge-0/0/1.0

# Enable IPv6 in IS-IS
set protocols isis topologies ipv6-unicast

# Apply metric to IPv6 topology on specific interface
set protocols isis interface ge-0/0/0.0 topologies ipv6-unicast metric 20
```

## Configuring MT-ISIS on FRRouting

```bash
vtysh
configure terminal

router isis CORE
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 !
 ! Enable IPv6 topology
 topology ipv6-unicast

! Enable IS-IS on interface with IPv6 metric
interface eth0
 ip router isis CORE
 ipv6 router isis CORE
 isis ipv6 metric 10

end
write memory
```

## Per-Interface IPv6 Metric

MT-ISIS allows setting different metrics for IPv4 and IPv6 on the same interface:

```text
! Cisco: Set different IPv4 and IPv6 IS-IS metrics
interface GigabitEthernet0/0
 isis metric 10              ! IPv4 metric
 isis ipv6 metric 20         ! IPv6 metric (different topology)
```

This allows IPv6 traffic to prefer different paths than IPv4 for traffic engineering.

## Verifying MT-ISIS

```text
! Cisco
Router# show isis topology ipv6

! Show IPv6 MT IS-IS database
Router# show isis database verbose | include IPv6

! Show IPv6 routes from IS-IS
Router# show ipv6 route isis
```

```bash
# FRRouting
vtysh -c "show isis topology ipv6-unicast"
vtysh -c "show ipv6 route isis"
```

## Transition Mode: Single-Topology to Multi-Topology

When migrating an existing single-topology IS-IS network to MT:

```text
! Cisco: Use transition mode to allow compatibility with non-MT routers
Router(config)# router isis
Router(config-router)# address-family ipv6
Router(config-router-af)# multi-topology transition   ! Advertise both ST and MT TLVs
```

## Summary

MT-ISIS (RFC 5120) uses topology ID 2 for IPv6 unicast, allowing independent SPF computation for IPv4 and IPv6. Configure it with `multi-topology` under the IPv6 address family on Cisco, `topology ipv6-unicast` on Juniper, or `topology ipv6-unicast` on FRRouting. Set per-topology interface metrics to engineer different forwarding paths for IPv4 and IPv6.
