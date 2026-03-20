# How to Configure BGP IPv4 Address Family Under the New Address-Family Model

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, Address Family, IPv4, Cisco IOS, Multiprotocol BGP, MPLS

Description: Learn how to configure BGP using the address-family model for IPv4 unicast, understanding when and why to use the explicit address-family syntax over the legacy flat model.

## The Old Model vs. the Address-Family Model

In older Cisco IOS BGP configurations, IPv4 neighbors were activated automatically and network statements lived directly under `router bgp`. The newer address-family model (required for multiprotocol BGP—MP-BGP) separates IPv4, IPv6, and VPN route families into explicit `address-family` blocks.

**Old model (legacy):**
```
router bgp 65001
 neighbor 10.0.0.1 remote-as 65002
 network 192.168.1.0 mask 255.255.255.0
```

**Address-family model (current standard):**
```
router bgp 65001
 neighbor 10.0.0.1 remote-as 65002
 !
 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  network 192.168.1.0 mask 255.255.255.0
 exit-address-family
```

## Step 1: Basic IPv4 Unicast Address Family Configuration

```
router bgp 65001
 bgp router-id 1.1.1.1

 ! Define neighbors (global section - no address family here)
 neighbor 203.0.113.1 remote-as 65100
 neighbor 203.0.113.1 description ISP1-eBGP
 neighbor 203.0.113.1 password BGPSecret

 ! IPv4 Unicast Address Family
 address-family ipv4 unicast
  ! Activate this neighbor for IPv4 unicast
  neighbor 203.0.113.1 activate

  ! Advertise networks
  network 198.51.100.0 mask 255.255.255.0

  ! Route filtering in address family
  neighbor 203.0.113.1 prefix-list BOGON_FILTER in
  neighbor 203.0.113.1 prefix-list OUR_PREFIX out

  ! Maximum prefixes
  neighbor 203.0.113.1 maximum-prefix 900000 80

 exit-address-family
```

## Step 2: Disable Auto-Summary

In the address-family model, disable automatic route summarization to prevent unexpected prefix aggregation:

```
router bgp 65001
 address-family ipv4 unicast
  ! Disable automatic summarization to classful boundaries
  no auto-summary

  ! Disable synchronization (not needed in modern networks)
  no synchronization
 exit-address-family
```

## Step 3: Configure Multiple Address Families on One Neighbor

The same neighbor can participate in multiple address families:

```
router bgp 65001
 neighbor 10.0.0.1 remote-as 65001   ! iBGP neighbor

 ! IPv4 Unicast - regular routing
 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
 exit-address-family

 ! IPv4 VRF - VPN routing (L3VPN)
 address-family ipv4 vrf CUSTOMER_A
  neighbor 10.0.0.2 remote-as 65500
  neighbor 10.0.0.2 activate
 exit-address-family

 ! IPv6 Unicast - if also doing IPv6 BGP
 address-family ipv6 unicast
  neighbor 10.0.0.1 activate
  network 2001:db8::/32
 exit-address-family
```

## Step 4: Configure the no bgp default ipv4-unicast Command

By default, Cisco IOS automatically activates neighbors in the IPv4 unicast address family. To require explicit activation (recommended for clarity):

```
router bgp 65001
 ! Require explicit activation - neighbors not active until added to address-family
 no bgp default ipv4-unicast

 neighbor 203.0.113.1 remote-as 65100
 ! Without 'no bgp default ipv4-unicast', neighbor is activated automatically
 ! With it, must explicitly activate:

 address-family ipv4 unicast
  neighbor 203.0.113.1 activate
 exit-address-family
```

This prevents accidental activation of neighbors in address families they shouldn't participate in.

## Step 5: Verify Address Family Configuration

```
! View active neighbors per address family
Router# show ip bgp ipv4 unicast summary

! View address family specific BGP table
Router# show ip bgp ipv4 unicast

! View neighbor address family capabilities
Router# show ip bgp neighbors 203.0.113.1 | include address family
```

## Step 6: Apply Route Maps in the Address Family

Route maps for filtering, attribute manipulation, and redistribution all go inside the address family:

```
router bgp 65001
 address-family ipv4 unicast
  ! Apply route map inbound from ISP
  neighbor 203.0.113.1 route-map INBOUND_POLICY in
  ! Apply route map outbound to ISP
  neighbor 203.0.113.1 route-map OUTBOUND_POLICY out

  ! Redistribute connected routes into BGP
  redistribute connected route-map CONNECTED_FILTER
 exit-address-family
```

## Conclusion

The BGP address-family model provides clear separation between routing protocols and address families, enabling multiprotocol BGP for IPv6, VPNs, and more. Use `no bgp default ipv4-unicast` for explicit neighbor activation, define all per-neighbor policies within the address-family block, and use `show ip bgp ipv4 unicast summary` to verify the address family is operational.
