# How to Configure IPv6 Router Advertisements on Cisco

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco, Router Advertisement, IOS, SLAAC, Networking

Description: Configure IPv6 Router Advertisements on Cisco IOS and IOS-XE routers to enable SLAAC address autoconfiguration and deliver DNS information to LAN clients.

## Introduction

Cisco routers natively support IPv6 Router Advertisements through the `ipv6 nd` (Neighbor Discovery) command set. This guide covers enabling RAs on Cisco IOS and IOS-XE, configuring prefix options, and delivering DNS information to clients.

## Prerequisites

- Cisco IOS 12.4(6)T or later, or any IOS-XE
- IPv6 unicast routing enabled globally
- Interface with an IPv6 address configured

## Enabling IPv6 Unicast Routing

Global IPv6 routing must be enabled before configuring RA on any interface:

```text
! Enable IPv6 unicast routing globally
Router(config)# ipv6 unicast-routing

! Verify it is enabled
Router# show ipv6 interface brief
```

## Configuring a Basic Router Advertisement

```text
! Assign an IPv6 address and enable RA on the LAN interface
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:db8:1:1::1/64
Router(config-if)# ipv6 nd ra-interval 100
Router(config-if)# ipv6 nd ra-lifetime 1800

! The interface sends RAs by default after ipv6 address is assigned
! Verify RA suppression is NOT active
Router(config-if)# no ipv6 nd ra suppress
```

## Configuring Prefix Advertisement Options

```text
! Control prefix options in the RA
Router(config)# interface GigabitEthernet0/0

! Set the prefix with custom lifetimes (valid-lft preferred-lft in seconds)
Router(config-if)# ipv6 nd prefix 2001:db8:1:1::/64 86400 14400

! Or advertise only the prefix without SLAAC (no-autoconfig)
Router(config-if)# ipv6 nd prefix 2001:db8:1:1::/64 86400 14400 no-autoconfig

! Set M and O flags to instruct clients to use DHCPv6
Router(config-if)# ipv6 nd managed-config-flag      ! M flag = 1
Router(config-if)# ipv6 nd other-config-flag         ! O flag = 1
```

## Configuring RDNSS (DNS via RA) on IOS-XE

IOS-XE 16.6+ supports advertising DNS servers directly in RA (RFC 8106):

```text
! Advertise a DNS server via Router Advertisement
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 nd ra dns server 2001:db8:1:1::53 infinite
Router(config-if)# ipv6 nd ra dns server 2606:4700:4700::1111 infinite

! Advertise DNS search domain (IOS-XE 16.9+)
Router(config-if)# ipv6 nd ra dns search-list corp.example.com infinite
```

## Configuring Router Preference

```text
! Set the default router preference (high, medium, or low)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 nd router-preference High

! This affects how clients prioritize this router vs. others on the same segment
```

## Suppressing Router Advertisements

For interfaces where RAs should not be sent (e.g., uplink/WAN interfaces):

```text
! Suppress RA on the WAN-facing interface
Router(config)# interface GigabitEthernet0/1
Router(config-if)# ipv6 nd ra suppress all
```

## Verifying Router Advertisement Configuration

```text
! Show RA configuration and statistics for an interface
Router# show ipv6 interface GigabitEthernet0/0

! Show ND prefix information
Router# show ipv6 nd interface GigabitEthernet0/0

! Show the IPv6 neighbor discovery table
Router# show ipv6 neighbors

! Debug RA messages in real time (use with caution in production)
Router# debug ipv6 nd
```

Sample output of `show ipv6 interface`:

```text
GigabitEthernet0/0 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::1
  Global unicast address(es):
    2001:DB8:1:1::1, subnet is 2001:DB8:1:1::/64
  ND DAD is enabled, number of DAD attempts: 1
  ND reachable time is 30000 milliseconds
  ND advertised reachable time is 0 (unspecified)
  ND advertised retransmit interval is 0 (unspecified)
  ND router advertisements are sent every 100 seconds
  ND router advertisements live for 1800 seconds
  ND advertised default router preference is Medium
```

## Conclusion

Cisco IOS and IOS-XE provide comprehensive IPv6 Router Advertisement configuration through the `ipv6 nd` command family. Key steps are enabling `ipv6 unicast-routing` globally, configuring a `/64` prefix on LAN interfaces, and tuning RA intervals and lifetimes. For modern deployments, use the RDNSS options on IOS-XE 16.6+ to deliver DNS server addresses without requiring a DHCPv6 server.
