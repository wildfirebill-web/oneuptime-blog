# How to Configure IPv6 on Cisco NX-OS Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco, NX-OS, Switch, Datacenter, Networking

Description: Configure IPv6 addressing, routing, and VRF support on Cisco NX-OS switches for datacenter deployments including SVIs, VRFs, and BGP with IPv6.

## Introduction

Cisco NX-OS is the operating system for Nexus series datacenter switches. While similar to IOS in syntax, NX-OS has some differences in IPv6 configuration, especially around features and VRF support. This guide covers IPv6 configuration for Nexus switches in a datacenter context.

## Step 1: Enable IPv6 Features

```
! Enable IPv6 unicast routing (enabled by default on most NX-OS versions)
N9K(config)# feature ipv6

! Enable OSPFv3 if needed
N9K(config)# feature ospfv3

! Enable BGP if needed (for datacenter fabrics)
N9K(config)# feature bgp
```

## Step 2: Configure IPv6 Addresses on Interfaces

```
! Configure a routed interface with IPv6
N9K(config)# interface Ethernet1/1
N9K(config-if)# no switchport
N9K(config-if)# ipv6 address 2001:db8:1:1::1/64
N9K(config-if)# ipv6 address fe80::1 link-local
N9K(config-if)# no shutdown

! Configure a Switch Virtual Interface (SVI) for a VLAN
N9K(config)# interface Vlan100
N9K(config-if)# ipv6 address 2001:db8:1:100::1/64
N9K(config-if)# no shutdown

! Configure a loopback (commonly used as router-id)
N9K(config)# interface Loopback0
N9K(config-if)# ipv6 address 2001:db8::1/128
```

## Step 3: Configure IPv6 in a VRF

VRFs are commonly used for tenant isolation in datacenter environments:

```
! Create a VRF and enable IPv6
N9K(config)# vrf context TENANT-A
N9K(config-vrf)# address-family ipv6 unicast

! Assign interface to VRF with IPv6 address
N9K(config)# interface Ethernet1/2
N9K(config-if)# no switchport
N9K(config-if)# vrf member TENANT-A
N9K(config-if)# ipv6 address 2001:db8:2:1::1/64
N9K(config-if)# no shutdown

! Static route in VRF
N9K(config)# ipv6 route 2001:db8:3::/48 2001:db8:2:1::2 vrf TENANT-A
```

## Step 4: Configure OSPFv3

```
! Configure OSPFv3 instance
N9K(config)# router ospfv3 CORE
N9K(config-router)# router-id 1.1.1.1

! Enable on interfaces
N9K(config)# interface Ethernet1/1
N9K(config-if)# ipv6 router ospfv3 CORE area 0

! Or in a VRF
N9K(config)# router ospfv3 TENANT
N9K(config-router)# vrf TENANT-A
N9K(config-router-vrf)# router-id 2.2.2.2
```

## Step 5: Configure BGP with IPv6

For spine-leaf fabrics using eBGP:

```
! Configure BGP with IPv6 address family
N9K(config)# router bgp 65001
N9K(config-router)# router-id 10.0.0.1
N9K(config-router)# address-family ipv6 unicast
N9K(config-router-af)# network 2001:db8::/48

! Configure BGP neighbor using IPv6
N9K(config-router)# neighbor 2001:db8:0:1::2 remote-as 65002
N9K(config-router-neighbor)# address-family ipv6 unicast
N9K(config-router-neighbor-af)# activate
```

## Step 6: Configure IPv6 ACL

```
! Create an IPv6 ACL
N9K(config)# ipv6 access-list IPV6-INBOUND
N9K(config-ipv6-acl)# permit icmp any any
N9K(config-ipv6-acl)# permit tcp any any established
N9K(config-ipv6-acl)# deny ipv6 any any log

! Apply to interface
N9K(config)# interface Ethernet1/1
N9K(config-if)# ipv6 traffic-filter IPV6-INBOUND in
```

## Verification Commands

```
! Show IPv6 interface status
N9K# show ipv6 interface brief

! Show IPv6 routing table
N9K# show ipv6 route

! Show IPv6 routing table for specific VRF
N9K# show ipv6 route vrf TENANT-A

! Show BGP IPv6 summary
N9K# show bgp ipv6 unicast summary

! Show OSPFv3 neighbors
N9K# show ipv6 ospf neighbor

! Show IPv6 neighbors (ARP equivalent)
N9K# show ipv6 neighbors

! Ping test
N9K# ping6 2001:db8::1 vrf TENANT-A
```

## Conclusion

Cisco NX-OS provides robust IPv6 support for datacenter deployments, with features like VRF-aware IPv6 routing, OSPFv3, and BGP with IPv6 address families. The syntax is largely similar to IOS with some NX-OS-specific differences like `feature ipv6` and VRF syntax. For large-scale datacenter fabrics, BGP with IPv6 underlay is the preferred routing protocol, providing scalable and simple configuration.
