# How to Configure IPv6 Underlay in Data Center Fabrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Underlay, Data Center, BGP, IS-IS, OSPF, Fabric Routing

Description: Configure IPv6 as the underlay routing protocol for data center fabrics using BGP unnumbered, IS-IS, or OSPFv3 to distribute loopback reachability for VXLAN and other overlay protocols.

---

The underlay is the physical routed network in a data center fabric. IPv6 underlay routing provides loopback reachability for overlay protocols (VXLAN, MPLS). BGP unnumbered, IS-IS with IPv6, or OSPFv3 are common choices, each with different complexity and feature trade-offs.

## BGP Unnumbered for IPv6 Underlay

```
BGP Unnumbered uses IPv6 link-local addresses for peering,
eliminating the need to allocate IPv4 or IPv6 P2P addresses.

Benefits:
- No address allocation for fabric links
- Auto-discovery of peers via IPv6 RA
- Works for both IPv4 and IPv6 underlay

Configuration (Cumulus Linux / FRR):
```

```bash
# /etc/frr/frr.conf - BGP Unnumbered on leaf

frr defaults datacenter

interface lo
 ipv6 address 2001:db8:dc1:200::101/128

interface swp1
 description Spine-1-Uplink
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra

interface swp2
 description Spine-2-Uplink
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra

router bgp 65101
 bgp router-id 10.0.1.101
 bgp bestpath as-path multipath-relax

 ! Unnumbered BGP - peers via IPv6 link-local
 neighbor fabric peer-group
 neighbor fabric remote-as external
 neighbor swp1 interface peer-group fabric
 neighbor swp2 interface peer-group fabric

 address-family ipv6 unicast
  neighbor fabric activate
  neighbor fabric next-hop-self
  network 2001:db8:dc1:200::101/128
  maximum-paths 64
 exit-address-family
```

## IS-IS IPv6 Underlay

```bash
# IS-IS with IPv6 - alternative to BGP for smaller fabrics
# /etc/frr/frr.conf - IS-IS IPv6

interface lo
 ipv6 address 2001:db8:dc1:200::101/128
 ip router isis DC-FABRIC
 isis passive

interface swp1
 ipv6 address 2001:db8:dc1:300:1:101::1/127
 ip router isis DC-FABRIC
 isis network point-to-point

router isis DC-FABRIC
 net 49.0001.0000.0101.0001.00   ! NET address (includes router ID)
 is-type level-2-only
 metric-style wide

 ! IPv6 in IS-IS
 address-family ipv6
  multi-topology
 exit-address-family

 log-adjacency-changes
```

```bash
# Cisco Nexus IS-IS IPv6 underlay
feature isis

router isis DC-UNDERLAY
  net 49.0001.0000.0001.0001.00
  is-type level-2
  address-family ipv6 unicast
    multi-topology
    maximum-paths 64

interface Ethernet1/1
  description Spine-1
  ipv6 address 2001:db8:dc1:300:1:101::1/127
  isis network point-to-point
  isis circuit-type level-2
  ip router isis DC-UNDERLAY
  ipv6 router isis DC-UNDERLAY
  no shutdown
```

## OSPFv3 IPv6 Underlay

```bash
# OSPFv3 for smaller fabrics (up to ~50 nodes)
# Cisco Nexus OSPFv3

feature ospfv3

router ospfv3 DC-UNDERLAY
  router-id 10.0.1.101
  log-adjacency-changes

interface loopback0
  ipv6 address 2001:db8:dc1:200::101/128
  ipv6 router ospfv3 DC-UNDERLAY area 0.0.0.0

interface Ethernet1/1
  description Spine-1
  ipv6 address 2001:db8:dc1:300:1:101::1/127
  ospfv3 network point-to-point
  ipv6 router ospfv3 DC-UNDERLAY area 0.0.0.0

! Verify
show ospfv3 neighbor
show ospfv3 database
show ipv6 route ospfv3
```

## VXLAN over IPv6 Underlay

```bash
# VXLAN VTEP with IPv6 underlay (VXLAN over IPv6)

# Cumulus Linux - VXLAN with IPv6 underlay
# /etc/network/interfaces

auto vxlan10
iface vxlan10
  vxlan-id 10
  vxlan-local-tunnelip 2001:db8:dc1:200::101  # IPv6 VTEP
  bridge-access 100

# For Arista EOS - VXLAN over IPv6 underlay
interface Vxlan1
   description VXLAN-IPv6-Underlay
   vxlan source-interface Loopback0  ! IPv6 loopback as VTEP
   vxlan vlan 100 vni 10100

# BGP EVPN over IPv6 underlay
router bgp 65101
   neighbor 2001:db8:dc1:200::1 remote-as 65001  ! Spine-1 IPv6 loopback
   address-family l2vpn evpn
      neighbor 2001:db8:dc1:200::1 activate
      advertise-all-vni
```

## Verify IPv6 Underlay

```bash
# Check IPv6 BGP/IS-IS adjacencies
show bgp ipv6 unicast summary
# or
show isis neighbor

# Verify IPv6 loopback reachability (critical for VXLAN)
show ipv6 route | grep "2001:db8:dc1:200"
# Should show all leaf loopbacks as /128 routes

# Test VTEP reachability via IPv6 underlay
ping6 2001:db8:dc1:200::102  # Leaf-2 loopback via IPv6 fabric

# Check VXLAN tunnels use IPv6 src/dst
show vxlan address-table
show vxlan vni 10100 vtep

# Verify ECMP is working for underlay
traceroute6 2001:db8:dc1:200::102
# Should show all paths through spines
```

IPv6 underlay in data center fabrics eliminates IPv4 dependency for fabric routing, with BGP unnumbered being the preferred approach as it uses IPv6 link-local addressing for peer discovery eliminating address allocation overhead, while IS-IS and OSPFv3 provide alternatives that converge faster and with less operational complexity for smaller fabrics.
