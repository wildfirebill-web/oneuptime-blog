# How to Configure OSPFv3 on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPFv3, Cisco, IPv6, Routing, IOS

Description: Step-by-step guide to configuring OSPFv3 for IPv6 routing on Cisco IOS and IOS-XE routers using the modern address-family syntax.

## Overview

Cisco supports OSPFv3 on IOS, IOS-XE, and IOS-XR. There are two configuration styles: the classic `ipv6 router ospf` syntax and the newer address-family syntax (recommended). This guide covers both.

## Prerequisites

```
! Verify IPv6 unicast routing is enabled
Router# show running-config | include ipv6 unicast-routing
ipv6 unicast-routing

! Enable if not present
Router(config)# ipv6 unicast-routing
```

## Classic OSPFv3 Configuration (IOS)

```
! Create the OSPFv3 process with process ID 1
Router(config)# ipv6 router ospf 1
Router(config-rtr)# router-id 1.1.1.1
Router(config-rtr)# exit

! Enable OSPFv3 on interfaces
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:db8:1::1/64
Router(config-if)# ipv6 ospf 1 area 0

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ipv6 address 2001:db8:2::1/64
Router(config-if)# ipv6 ospf 1 area 0
```

## Modern Address-Family Syntax (IOS-XE, Recommended)

The `ospfv3` address-family syntax is preferred on modern IOS-XE:

```
! Configure OSPFv3 with address-family syntax
Router(config)# router ospfv3 1
Router(config-router)# router-id 1.1.1.1
Router(config-router)# address-family ipv6 unicast
Router(config-router-af)# exit
Router(config-router)# exit

! Enable on interfaces using ospfv3 command
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ospfv3 1 ipv6 area 0

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ospfv3 1 ipv6 area 0
```

## Setting Interface Cost

```
! Set OSPFv3 cost on an interface (lower = preferred)
Router(config-if)# ospfv3 1 ipv6 cost 10

! Set hello and dead intervals
Router(config-if)# ospfv3 1 ipv6 hello-interval 10
Router(config-if)# ospfv3 1 ipv6 dead-interval 40
```

## Configuring a Passive Interface

Prevent OSPFv3 Hello packets from being sent on a stub interface (e.g., loopback):

```
Router(config)# router ospfv3 1
Router(config-router)# address-family ipv6 unicast
Router(config-router-af)# passive-interface Loopback0
```

## Verification Commands

```
! Show OSPFv3 neighbor state
Router# show ospfv3 neighbor

! Show OSPFv3 interfaces
Router# show ospfv3 interface brief

! Show OSPFv3 Link State Database
Router# show ospfv3 database

! Show IPv6 routes learned via OSPFv3
Router# show ipv6 route ospf

! Verify adjacency with specific neighbor
Router# show ospfv3 neighbor detail
```

## Sample Output

```
Router# show ospfv3 neighbor

OSPFv3 1 address-family ipv6 (router-id 1.1.1.1)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
2.2.2.2           1   FULL/BDR        00:00:37    3               GigabitEthernet0/0
3.3.3.3           1   FULL/  -        00:00:36    4               GigabitEthernet0/1
```

## Troubleshooting Adjacency Issues

```
! Enable OSPFv3 debugging (caution: verbose output)
Router# debug ospfv3 adj

! Check for Hello timer mismatch
Router# show ospfv3 interface GigabitEthernet0/0

! Verify the area matches on both routers
Router# show ospfv3 database router
```

## Summary

Cisco OSPFv3 configuration requires enabling `ipv6 unicast-routing`, setting a Router ID, and enabling OSPFv3 per interface with `ospfv3 1 ipv6 area 0`. Use the modern address-family syntax on IOS-XE. Verify with `show ospfv3 neighbor` and confirm IPv6 routes in `show ipv6 route ospf`.
