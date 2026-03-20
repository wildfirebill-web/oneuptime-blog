# How to Configure IPv6 Firewall Rules on Cisco ASA

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco ASA, Firewall, Access Control, Enterprise

Description: Learn how to configure IPv6 firewall policies on Cisco ASA, including interface security levels, IPv6 ACLs, inspection policies, and stateful connection tracking.

## Overview

Cisco ASA supports IPv6 firewalling with stateful inspection. ASA uses security levels on interfaces (0=outside, 100=inside) to automatically permit traffic from higher to lower security levels while blocking the reverse. For IPv6, explicit ACLs can override or supplement the security level policy. IPv6 must be explicitly enabled on each interface.

## Enable IPv6 on ASA Interfaces

```
! Enable IPv6 globally
ipv6 unicast-routing

! Configure IPv6 on interfaces
interface GigabitEthernet0/0
 description "Internet (Outside)"
 nameif outside
 security-level 0
 ipv6 address 2001:db8:wan::1/64
 ipv6 enable
 no shutdown

interface GigabitEthernet0/1
 description "Internal LAN (Inside)"
 nameif inside
 security-level 100
 ipv6 address 2001:db8:lan::1/64
 ipv6 enable
 no shutdown
```

## IPv6 Access Control Lists

### Create IPv6 ACL for Inbound Traffic

```
! IPv6 ACL for outside interface (inbound = from internet)
ipv6 access-list OUTSIDE-IN-V6

 ! Allow established TCP (handled by stateful engine, but explicit for clarity)
 permit tcp any any established

 ! Allow essential ICMPv6 — NEVER block these
 permit icmp6 any any unreachable             ! Type 1
 permit icmp6 any any packet-too-big          ! Type 2 — NEVER BLOCK
 permit icmp6 any any time-exceeded           ! Type 3
 permit icmp6 any any parameter-problem       ! Type 4

 ! Allow NDP from link-local only
 permit icmp6 fe80::/10 any neighbor-solicitation
 permit icmp6 fe80::/10 any neighbor-advertisement
 permit icmp6 fe80::/10 any router-solicitation
 permit icmp6 fe80::/10 any router-advertisement

 ! Allow HTTPS inbound
 permit tcp any host 2001:db8:lan::web eq 443

 ! Allow SSH from management
 permit tcp fd00:mgmt::/48 host 2001:db8:lan::1 eq 22

 ! Implicit deny (any any)
 deny ipv6 any any

! Apply ACL to outside interface
access-group OUTSIDE-IN-V6 in interface outside
```

## ICMPv6 Inspection Policy

```
! Enable ICMPv6 stateful inspection
! This allows ICMP error messages related to established TCP/UDP flows
policy-map global_policy
 class inspection_default
  inspect icmp

! Or create dedicated ICMPv6 inspection:
class-map ICMPV6-INSPECT
 match protocol icmp6

policy-map ICMPv6-POLICY
 class ICMPV6-INSPECT
  inspect icmp

service-policy ICMPv6-POLICY interface outside
```

## Stateful IPv6 Inspection

ASA tracks IPv6 connections in its state table:

```
! View active IPv6 connections
show conn ipv6

! Sample output:
!   TCP outside  2001:db8:client::1:54321 inside 2001:db8:srv::1:443, idle 0:00:05, bytes 1234

! View IPv6 connection counts
show conn count

! View IPv6 routing
show ipv6 route

! Debug IPv6 packet flow
packet-tracer input outside ipv6 2001:db8:ext::1 tcp 54321 2001:db8:srv::1 443 detailed
```

## IPv6 Object Groups

For cleaner ACL management:

```
! Create IPv6 object group for management hosts
object-group network IPV6-MANAGEMENT
 network-object fd00:mgmt::/48
 network-object 2001:db8:admin::1/128
 description "IPv6 Management Networks"

! Create object group for internal servers
object-group network IPV6-SERVERS
 network-object host 2001:db8:lan::web
 network-object host 2001:db8:lan::mail

! Use in ACL
ipv6 access-list OUTSIDE-IN-V6
 permit tcp object-group-network IPV6-MANAGEMENT any eq 22
 permit tcp any object-group-network IPV6-SERVERS eq 443
```

## IPv6 Connection Timeout and Limits

```
! Configure IPv6 connection timeouts
timeout conn 1:00:00

! Limit TCP embryonic (half-open) connections per host
object network IPV6-SERVERS
 host 2001:db8:lan::web
service-policy global_policy interface outside
! (Configure connection limits in service policy)
```

## Bogon Filtering

```
! Block documentation prefix from internet
ipv6 access-list BOGON-BLOCK-V6
 deny ipv6 2001:db8::/32 any
 deny ipv6 ::/128 any
 deny ipv6 ::1/128 any
 deny ipv6 fc00::/7 any
 permit ipv6 any any

access-group BOGON-BLOCK-V6 in interface outside
```

## Verification

```
! Show IPv6 interface status
show ipv6 interface

! Show IPv6 ACL counters
show ipv6 access-list OUTSIDE-IN-V6

! Show IPv6 connections
show conn ipv6 detail

! Test packet traversal (packet tracer)
packet-tracer input outside ipv6 2001:db8:attacker::1 tcp 12345 2001:db8:srv::1 22 detailed
! Shows: whether the packet would be allowed or dropped and why
```

## Summary

Cisco ASA IPv6 firewalling uses security levels (inside=100, outside=0) for default policy and explicit ACLs for granular control. Enable IPv6 with `ipv6 unicast-routing` and `ipv6 enable` on each interface. Create IPv6 ACLs with `ipv6 access-list NAME` and attach with `access-group NAME in interface outside`. Always allow ICMPv6 Packet Too Big (type 2) — ASA may block it by default if not explicitly permitted. Use `packet-tracer` to test how specific IPv6 packets would be handled without actually sending them. Use object groups for manageable ACL definitions.
