# How to Redistribute IPv6 Routes into IS-IS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, Route Redistribution, Routing, Networking

Description: Learn how to redistribute IPv6 static, connected, OSPFv3, and BGP routes into IS-IS on Cisco, Juniper, and FRRouting.

## Overview

IS-IS route redistribution advertises routes from external sources as External routes in the IS-IS LSP database. These are marked as External Level-2 routes and have a higher administrative distance than internal IS-IS routes.

## Cisco IOS Redistribution

```text
! Redistribute static IPv6 routes into IS-IS
Router(config)# router isis
Router(config-router)# redistribute ipv6 static level-2   ! Inject into L2

! Redistribute connected IPv6 routes
Router(config-router)# redistribute ipv6 connected level-2

! Redistribute OSPFv3 routes
Router(config-router)# redistribute ipv6 ospf 1 level-2 metric 20

! Redistribute BGP routes
Router(config-router)# redistribute ipv6 bgp 65001 level-2 metric 30
```

## Cisco: Using Route Maps for Selective Redistribution

```text
! Create prefix list for filtering
Router(config)# ipv6 prefix-list ISIS_IMPORT seq 10 permit 2001:db8:branch::/48

! Create route map
Router(config)# route-map TO_ISIS permit 10
Router(config-route-map)#  match ipv6 address prefix-list ISIS_IMPORT
Router(config-route-map)#  set metric 15

! Apply to redistribution
Router(config)# router isis
Router(config-router)# redistribute ipv6 static level-2 route-map TO_ISIS
```

## Juniper JunOS Redistribution

```bash
# Redistribution uses export policies in Juniper

# Policy to redistribute static IPv6 into IS-IS

set policy-options policy-statement STATIC_TO_ISIS term 1 from protocol static
set policy-options policy-statement STATIC_TO_ISIS term 1 from family inet6
set policy-options policy-statement STATIC_TO_ISIS term 1 then accept

# Policy for connected routes
set policy-options policy-statement CONNECTED_TO_ISIS term 1 from protocol direct
set policy-options policy-statement CONNECTED_TO_ISIS term 1 from family inet6
set policy-options policy-statement CONNECTED_TO_ISIS term 1 then accept

# Apply export policy to IS-IS
set protocols isis export STATIC_TO_ISIS
set protocols isis export CONNECTED_TO_ISIS
```

## FRRouting Redistribution

```bash
vtysh
configure terminal

router isis CORE

 ! Redistribute connected routes
 redistribute ipv6 connected
 redistribute ipv6 static

 ! Redistribute OSPFv3 routes into IS-IS
 redistribute ipv6 ospf6

 ! Redistribute BGP routes
 redistribute ipv6 bgp

end
write memory
```

## Setting Default Metric for Redistributed Routes

```bash
# FRRouting
router isis CORE
 default-information originate ipv6 always metric 100   ! Advertise default route
 metric-style wide    ! Required for wide metrics

# Cisco
Router(config)# router isis
Router(config-router)# default-information originate ipv6 always
```

## Verifying Redistributed Routes

```text
! Cisco: Check for external IS-IS routes on a neighbor
Router-Neighbor# show ipv6 route isis

I2 EX 2001:DB8:BRANCH::/48 [115/20]  ← External IS-IS route
     via FE80::asbr, GigabitEthernet0/0
```

```bash
# FRRouting: Check external routes
vtysh -c "show ipv6 route isis" | grep "EX\|extern"

# Juniper
show route protocol isis table inet6.0
# External routes will show metric type: External
```

## Preventing Route Loops

When redistributing between two protocols bidirectionally, use route tags to prevent loops:

```text
! Cisco: Tag routes from OSPF before injecting into IS-IS
Router(config)# route-map OSPF_TO_ISIS permit 10
Router(config-route-map)#  match ipv6 address prefix-list OSPF_PREFIXES
Router(config-route-map)#  set tag 100

! In OSPFv3, block routes tagged with 100 from being redistributed back
Router(config)# route-map ISIS_TO_OSPF deny 5
Router(config-route-map)#  match tag 100

Router(config)# route-map ISIS_TO_OSPF permit 10
```

## Summary

IS-IS IPv6 redistribution is configured with `redistribute ipv6 <protocol> level-<n>` on Cisco, export policies on Juniper, and `redistribute ipv6 <protocol>` on FRRouting. External IS-IS routes appear with an external flag in the routing table. Always use route maps or policies for selective redistribution and prevent bidirectional redistribution loops using route tags.
