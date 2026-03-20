# How to Redistribute Routes into EIGRPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: EIGRPv6, IPv6, Route Redistribution, Cisco, Routing

Description: Learn how to redistribute IPv6 static, connected, OSPFv3, and BGP routes into EIGRPv6 on Cisco IOS with metric control and route maps.

## Overview

Route redistribution into EIGRPv6 requires specifying a metric (or seed metric) since EIGRP uses a composite metric. Without a metric, redistributed routes are not installed (default metric is 0, which EIGRP treats as unreachable).

## Setting a Default Metric for Redistribution

The safest approach is to set a default metric that applies to all redistributed routes:

```
! Classic EIGRPv6
Router(config)# ipv6 router eigrp 1
Router(config-rtr)# default-metric 10000 100 255 1 1500
! Format: bandwidth(kbps) delay(microseconds) reliability load MTU

! Named EIGRPv6
Router(config)# router eigrp MY_NETWORK
Router(config-router)# address-family ipv6 unicast autonomous-system 1
Router(config-router-af)# default-metric 10000 100 255 1 1500
```

## Redistributing Static Routes

```
! Classic EIGRPv6
Router(config)# ipv6 router eigrp 1
Router(config-rtr)# redistribute static metric 10000 100 255 1 1500

! Named EIGRPv6
Router(config)# router eigrp MY_NETWORK
Router(config-router)# address-family ipv6 unicast autonomous-system 1
Router(config-router-af)# redistribute static metric 10000 100 255 1 1500
```

## Redistributing Connected Routes

```
! Classic EIGRPv6
Router(config)# ipv6 router eigrp 1
Router(config-rtr)# redistribute connected metric 1000000 10 255 1 1500
```

## Redistributing from OSPFv3

```
! Classic EIGRPv6 — redistribute OSPF process 1
Router(config)# ipv6 router eigrp 1
Router(config-rtr)# redistribute ospf 1 metric 10000 100 255 1 1500
Router(config-rtr)# redistribute ospf 1 include-connected  ! Also include connected subnets
```

## Redistributing from BGP

```
! Named EIGRPv6 — redistribute BGP
Router(config)# router eigrp MY_NETWORK
Router(config-router)# address-family ipv6 unicast autonomous-system 1
Router(config-router-af)# redistribute bgp 65001 metric 10000 1000 255 1 1500
```

## Using Route Maps for Selective Redistribution

```
! Create a prefix list for filtering
Router(config)# ipv6 prefix-list EIGRP_IMPORT seq 10 permit 2001:db8:branch::/48

! Create a route map
Router(config)# route-map OSPF_TO_EIGRP permit 10
Router(config-route-map)#  match ipv6 address prefix-list EIGRP_IMPORT
Router(config-route-map)#  set metric 10000 100 255 1 1500   ! Override default metric

Router(config)# ipv6 router eigrp 1
Router(config-rtr)# redistribute ospf 1 route-map OSPF_TO_EIGRP
```

## Understanding the EIGRP Metric Components

The 5-part metric specification: `<bandwidth> <delay> <reliability> <load> <MTU>`

| Parameter | Unit | Typical Value | Notes |
|-----------|------|---------------|-------|
| Bandwidth | Kbps | 10000 (10 Mbps) | Min bandwidth on the path |
| Delay | Microseconds | 100 | Cumulative delay |
| Reliability | 1-255 | 255 | 255 = 100% reliable |
| Load | 1-255 | 1 | 1 = minimum load |
| MTU | Bytes | 1500 | Not used in metric calc |

## Verifying Redistributed Routes

```
! Check that redistributed routes appear as "D EX" in the routing table
Router-Neighbor# show ipv6 route eigrp

D EX 2001:DB8:BRANCH::/48 [170/30720]   ← External EIGRP route (AD=170)
     via FE80::2, GigabitEthernet0/0

! External EIGRP routes have AD 170 (vs 90 for internal)

! On the redistributing router
Router# show ipv6 eigrp topology | include EX
```

## Summary

Redistributing into EIGRPv6 requires a 5-part seed metric. Use `default-metric` to set a global default, or specify the metric per `redistribute` statement. Redistributed routes appear as `D EX` with administrative distance 170. Use route maps to selectively redistribute specific prefixes and prevent route leaks between routing domains.
