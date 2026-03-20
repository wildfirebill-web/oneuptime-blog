# How to Configure BGP Origin Validation for IPv6 on Cisco

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, RPKI, IPv6, Cisco, Routing Security

Description: Configure RPKI-based BGP origin validation for IPv6 prefixes on Cisco IOS-XE and IOS-XR routers to reject invalid route announcements.

## Overview

Cisco IOS-XE (15.2+) and IOS-XR support BGP origin validation via RPKI. Routers connect to an RPKI cache server using the RTR protocol and use ROA data to validate incoming IPv6 BGP prefixes.

## Prerequisites

- Cisco IOS-XE 15.2+ or IOS-XR 5.3+
- An RPKI validator (Routinator, RIPE Validator) accessible from the router
- IPv6 BGP sessions already configured

## Step 1: Configure the RPKI Cache Server

```
! IOS-XE: Configure RPKI cache (RTR) server
router bgp 64496
 bgp rpki server tcp 192.0.2.100 port 3323 refresh 600
 !
 ! If validator is reachable via IPv6
 bgp rpki server tcp 2001:db8:validator::1 port 3323 refresh 600
```

## Step 2: Verify RPKI Cache Connection

```
! Check RPKI cache server status
show bgp rpki server

! Expected output:
! BGP RPKI cache servers:
! Server: 2001:db8:validator::1:3323
!   State: Connected
!   Uptime: 00:15:43
!   ROAs: 300000 IPv4, 85000 IPv6
```

## Step 3: Enable BGP Origin Validation

```
! Enable origin validation for BGP
router bgp 64496
 bgp origin-validation signal ibgp
 !
 ! Enable for IPv6 address family
 address-family ipv6 unicast
  bgp origin-validation signal ibgp
 exit-address-family
```

## Step 4: Configure Route Maps to Act on Validation State

```
! Define route-maps for each validation state
route-map RPKI-POLICY permit 10
 match rpki valid
 set local-preference 200
!
route-map RPKI-POLICY permit 20
 match rpki not-found
 set local-preference 100
!
! Deny INVALID routes (omit or add deny statement)
route-map RPKI-POLICY deny 30
 match rpki invalid
!
route-map RPKI-POLICY permit 40

! Apply to BGP neighbor
router bgp 64496
 neighbor 2001:db8:peer::1 remote-as 65001
 address-family ipv6 unicast
  neighbor 2001:db8:peer::1 route-map RPKI-POLICY in
 exit-address-family
```

## Step 5: Verify Origin Validation Status

```
! Show BGP IPv6 table with OV (Origin Validation) status
show bgp ipv6 unicast

! Look for columns: Status codes include 'V' (Valid), 'I' (Invalid), '?' (Not found)

! Check a specific prefix
show bgp ipv6 unicast 2001:db8::/32

! Show all INVALID IPv6 routes
show bgp ipv6 unicast rpki invalid

! Show all VALID IPv6 routes
show bgp ipv6 unicast rpki valid
```

## Step 6: IOS-XR Configuration

On Cisco IOS-XR, the configuration syntax differs:

```
! IOS-XR RPKI configuration
router bgp 64496
 rpki server 2001:db8:validator::1
  transport tcp port 3323
  refresh-time 600
 !

 ! Apply validation in routing policy
 route-policy RPKI-VALIDATION
   if validation-state is valid then
     set local-preference 200
   elseif validation-state is not-found then
     set local-preference 100
   else
     drop
   endif
 end-policy

 neighbor 2001:db8:peer::1
  remote-as 65001
  address-family ipv6 unicast
   route-policy RPKI-VALIDATION in
  !
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your Cisco routers' BGP sessions and RPKI validator connectivity. Set up SNMP or API-based monitors to alert on RPKI cache disconnections.

## Conclusion

BGP origin validation on Cisco involves connecting to an RPKI cache server, enabling validation for IPv6 address families, and applying route-maps to act on VALID/INVALID/NOT-FOUND states. Start with preferring valid routes before dropping invalid ones to minimize disruption.
