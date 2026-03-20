# How to Configure BGP IPv6 Route Policies and Filters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Route Policy, Filtering, Routing

Description: Learn how to configure BGP IPv6 inbound and outbound route policies using prefix lists and route maps to control prefix advertisement and acceptance.

## Overview

BGP route policies and filters are essential security and operational controls. Without them, a BGP peer could advertise millions of routes or hijack your prefixes. This guide covers inbound and outbound filtering with prefix lists and route maps on FRRouting and Cisco.

## Why Filtering is Critical

- **Inbound filtering**: Accept only the prefixes your peer is authorized to announce
- **Outbound filtering**: Send only the prefixes you own to prevent route leaks
- **Anti-hijack protection**: Reject default routes and bogon prefixes from peers

## FRRouting: Basic Prefix List Filtering

```bash
vtysh
configure terminal

! Inbound: accept only the peer's allocated space
ipv6 prefix-list PEER_IN seq 5  permit 2001:db8:peer::/48 le 64
ipv6 prefix-list PEER_IN seq 99 deny ::/0 le 128

! Outbound: advertise only our own prefix
ipv6 prefix-list PEER_OUT seq 5  permit 2001:db8:mynet::/48
ipv6 prefix-list PEER_OUT seq 99 deny ::/0 le 128

router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8:peer::2 prefix-list PEER_IN  in
  neighbor 2001:db8:peer::2 prefix-list PEER_OUT out
 exit-address-family

end
write memory
```

## FRRouting: Route Maps for Advanced Policy

Route maps allow more granular control, including setting LOCAL_PREF, MED, and communities:

```bash
vtysh
configure terminal

! Deny bogon prefixes
ipv6 prefix-list BOGONS seq 5  deny  ::/0
ipv6 prefix-list BOGONS seq 10 deny  ::1/128
ipv6 prefix-list BOGONS seq 15 deny  fc00::/7 le 128     ! ULA
ipv6 prefix-list BOGONS seq 20 deny  fe80::/10 le 128    ! Link-local
ipv6 prefix-list BOGONS seq 25 deny  fec0::/10 le 128    ! Deprecated site-local
ipv6 prefix-list BOGONS seq 30 deny  ff00::/8  le 128    ! Multicast
ipv6 prefix-list BOGONS seq 99 permit ::/0 le 128

! Route map: set local preference for preferred peer
route-map PREFER_PEER permit 10
 match ipv6 address prefix-list BOGONS
 set local-preference 150

route-map PREFER_PEER permit 20
 match ipv6 address prefix-list BOGONS
 set local-preference 100

router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8:peer::2 route-map PREFER_PEER in
 exit-address-family

end
```

## Cisco: Prefix List and Route Map Filtering

```text
! Create IPv6 prefix lists
Router(config)# ipv6 prefix-list PEER_IN  seq 5  permit 2001:db8:peer::/48 le 64
Router(config)# ipv6 prefix-list PEER_IN  seq 99 deny ::/0 le 128
Router(config)# ipv6 prefix-list PEER_OUT seq 5  permit 2001:db8:mynet::/48
Router(config)# ipv6 prefix-list PEER_OUT seq 99 deny ::/0 le 128

! Apply to BGP neighbor
Router(config)# router bgp 65001
Router(config-router)# address-family ipv6 unicast
Router(config-router-af)# neighbor 2001:db8:peer::2 prefix-list PEER_IN  in
Router(config-router-af)# neighbor 2001:db8:peer::2 prefix-list PEER_OUT out
```

## Maximum Prefix Limit (DoS Protection)

Limit the number of prefixes accepted from a peer to prevent BGP table flooding:

```bash
# FRRouting: limit prefixes from a peer

router bgp 65001
 address-family ipv6 unicast
  ! Accept maximum 1000 prefixes; warn at 80%; tear down session at limit
  neighbor 2001:db8:peer::2 maximum-prefix 1000 80 restart 5
 exit-address-family
```

```text
! Cisco
Router(config-router-af)# neighbor 2001:db8:peer::2 maximum-prefix 1000 80 restart 5
```

## Soft Reconfiguration for Policy Changes

After changing a policy, reset the BGP session softly to apply new filters without tearing down the session:

```bash
# Apply new inbound policy without resetting session
vtysh -c "clear bgp ipv6 unicast 2001:db8:peer::2 soft in"

# Apply new outbound policy
vtysh -c "clear bgp ipv6 unicast 2001:db8:peer::2 soft out"
```

## Summary

Always configure both inbound and outbound BGP IPv6 filters. Use prefix lists for simple allow/deny rules and route maps for complex policies involving LOCAL_PREF, MED, or community tagging. Implement maximum-prefix limits on all eBGP sessions to prevent route floods from misconfigured peers.
