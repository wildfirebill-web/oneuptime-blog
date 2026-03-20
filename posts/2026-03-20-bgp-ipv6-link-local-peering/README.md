# How to Peer BGP Over IPv6 Link-Local Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Link-Local, Fe80, Peering

Description: Learn how to configure BGP peering over IPv6 link-local addresses for direct connected peers, including the interface requirement and next-hop handling.

## Overview

BGP can peer using IPv6 link-local addresses (fe80::/10) when two routers are directly connected on the same link. This is common in data center fabrics (BGP unnumbered), IXP route servers, and lab environments. Link-local peering eliminates the need to assign global addresses to router interconnect links.

## Why Use Link-Local BGP Peering?

- **Simplified addressing** - No need to allocate global /64 or /126 for each router link
- **Automatic availability** - Link-local addresses are always present on IPv6 interfaces
- **Unnumbered interfaces** - Common in modern data center spine-leaf designs
- **Standard at IXPs** - Many IXPs support link-local BGP peering on route servers

## Configuration on FRRouting

```bash
vtysh
configure terminal

router bgp 65001
 bgp router-id 1.1.1.1

 ! Peer using link-local address - MUST specify interface
 neighbor fe80::2%eth0 remote-as 65002
 neighbor fe80::2%eth0 interface eth0     ! Required for link-local peers
 neighbor fe80::2%eth0 description "Link-local BGP peer on eth0"

 address-family ipv6 unicast
  neighbor fe80::2%eth0 activate
  network 2001:db8:1::/48
 exit-address-family

end
write memory
```

## Configuration on Cisco IOS-XE (BGP Unnumbered)

```text
! Enable BGP peering on an unnumbered interface
interface GigabitEthernet0/0
 ipv6 address fe80::1 link-local
 ipv6 enable

router bgp 65001
 bgp router-id 1.1.1.1

 ! Use interface name instead of IP address for unnumbered BGP
 neighbor GigabitEthernet0/0 interface remote-as 65002

 address-family ipv6 unicast
  neighbor GigabitEthernet0/0 activate
  network 2001:db8:1::/48
 exit-address-family
```

## Next-Hop Handling

When BGP receives a route with a link-local next hop from a link-local peer, the next hop is only valid on that specific link. This creates a challenge for iBGP route reflection:

```bash
# FRRouting - next-hop-self required for iBGP when link-local peers exist

router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8::route-reflector next-hop-self    # Replace link-local NH
 exit-address-family
```

## Verifying Link-Local BGP Sessions

```bash
# FRRouting - check peer state
vtysh -c "show bgp ipv6 unicast summary"

# Neighbor should show with link-local address
# fe80::2       4   65002   100   100   5   0   0  00:45:12  8

# Show neighbor details
vtysh -c "show bgp neighbors fe80::2%eth0"
# Look for: BGP state = Established
```

## Troubleshooting Link-Local BGP

```bash
# Verify link-local address is configured on the interface
ip -6 addr show dev eth0 | grep "scope link"

# Verify the peer's link-local address is in the neighbor table
ip -6 neigh show dev eth0 | grep "fe80::2"

# If the neighbor is not discovered:
ping6 fe80::2%eth0   # Test reachability first

# Capture BGP OPEN messages
sudo tcpdump -i eth0 -n "tcp port 179"
```

## Multi-Hop Link-Local BGP (Not Supported)

Link-local addresses are not routable beyond the local link. Therefore, **link-local BGP peering only works for directly connected (single-hop) peers**. For multi-hop eBGP, use global unicast addresses.

## Summary

BGP link-local peering uses fe80:: addresses and requires specifying the interface in the neighbor configuration (e.g., `fe80::2%eth0` in FRRouting). It is ideal for directly connected peers and BGP unnumbered data center deployments. Always use `next-hop-self` when reflecting link-local routes to iBGP peers that cannot reach the link-local next hop.
