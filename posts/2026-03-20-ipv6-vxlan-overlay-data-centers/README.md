# How to Configure IPv6 VXLAN Overlay in Data Centers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VXLAN, Overlay, Data Center, EVPN, BGP, Network Virtualization

Description: Configure IPv6 VXLAN overlays in data centers including VXLAN transport over IPv6 underlay, BGP EVPN with IPv6 address families, and IPv6 host mobility across VTEP boundaries.

---

VXLAN (Virtual Extensible LAN) creates overlay Layer 2 networks over Layer 3 underlay. IPv6 VXLAN can run over IPv4 or IPv6 underlay, and carry IPv6 host traffic in the overlay. BGP EVPN (Ethernet VPN) distributes VTEP and MAC/IP bindings for optimal forwarding.

## VXLAN IPv6 Architecture

```
VXLAN IPv6 Overlay Components:

Underlay (IPv4 or IPv6 fabric):
  Spine/Leaf routing (BGP/IS-IS)
  VTEP loopback reachability

Overlay (VXLAN encapsulation):
  VNI 10100 → VLAN 100 (Tenant A workloads)
  VNI 10200 → VLAN 200 (Tenant B workloads)

Host addressing (inside VXLAN overlay):
  VLAN 100: 2001:db8:tenant-a::/64
  VLAN 200: 2001:db8:tenant-b::/64

VXLAN Packet Structure:
[IPv6 Outer Header][UDP 4789][VXLAN Header][Inner Ethernet][IPv6 Payload]
 ← Underlay: Leaf loopbacks → ←  Overlay: Host IPv6 addresses →
```

## VXLAN VTEP Configuration (Linux/FRR)

```bash
# Cumulus Linux - VXLAN with IPv6 hosts

# Configure VTEP
# /etc/network/interfaces

auto lo
iface lo inet loopback
  address 10.0.1.101/32
  # Also configure IPv6 loopback for underlay
  address6 2001:db8:dc1:200::101/128

# VXLAN interface (carries IPv6 host traffic)
auto vxlan100
iface vxlan100
  vxlan-id 100
  vxlan-local-tunnelip 10.0.1.101  # Use IPv4 loopback for VTEP
  # Or for IPv6 underlay:
  # vxlan-local-tunnelip 2001:db8:dc1:200::101

auto bridge
iface bridge
  bridge-ports swp5 swp6 vxlan100  # Server ports + VXLAN
  bridge-vids 100

# SVI for IPv6 routing in VXLAN segment
auto vlan100
iface vlan100
  address 2001:db8:tenant-a::1/64
  address-virtual 00:00:5e:00:01:01 2001:db8:tenant-a::1/64
  vlan-raw-device bridge
  vlan-id 100
```

## BGP EVPN for IPv6 VXLAN

```bash
# /etc/frr/frr.conf - BGP EVPN with IPv6 host routes

frr defaults datacenter

router bgp 65101
  bgp router-id 10.0.1.101

  ! Underlay BGP (ipv4 or ipv6 peering)
  neighbor SPINES peer-group
  neighbor SPINES remote-as external

  ! EVPN address family for VXLAN
  address-family l2vpn evpn
    neighbor SPINES activate
    advertise-all-vni
    !
    ! Advertise IPv6 host routes (Type 2 MAC/IP routes)
    advertise ipv6 unicast
  exit-address-family

  ! IPv6 global routing
  address-family ipv6 unicast
    neighbor SPINES activate
    redistribute connected
  exit-address-family
```

## Arista EOS VXLAN IPv6 Configuration

```
! Arista EOS - VXLAN with IPv6 hosts

! VTEP source interface
interface Loopback0
   ip address 10.0.1.101/32
   ipv6 address 2001:db8:dc1:200::101/128

interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200

! SVI with anycast gateway for IPv6 hosts
interface Vlan100
   ipv6 address 2001:db8:tenant-a::1/64
   ip virtual-router address ipv6 2001:db8:tenant-a::1

! BGP EVPN
router bgp 65101
   !
   address-family evpn
      neighbor SPINE activate
      !
   address-family ipv6
      redistribute connected

! Verify EVPN routes
show bgp evpn route-type mac-ip
! Should show: IPv6 addresses bound to MAC addresses
! Type 2 routes: MAC + IPv6 host /128

show bgp evpn route-type prefix
! IPv6 prefix routes for L3 routing
```

## IPv6 Host Mobility in VXLAN

```bash
# When VM migrates from Leaf-1 to Leaf-2:

# Old VTEP (Leaf-1) withdraws MAC/IP:
# EVPN Type 2: withdraw 2001:db8:tenant-a::100, MAC aa:bb:cc:dd:ee:ff, VNI 10100

# New VTEP (Leaf-2) advertises:
# EVPN Type 2: announce 2001:db8:tenant-a::100, MAC aa:bb:cc:dd:ee:ff, VNI 10100

# All VTEPs update their MAC tables automatically via BGP EVPN
# IPv6 traffic redirects to new VTEP within BGP convergence time

# Monitor EVPN updates
show bgp evpn neighbor <spine-ip> routes
# Watch for Type 2 route withdrawals/advertisements

# Check local MAC/IP binding
show mac address-table
show bgp evpn route-type mac-ip detail | grep 2001:db8:tenant-a
```

## NDP Suppression for IPv6 VXLAN

```bash
# EVPN NDP suppression: Leaves answer IPv6 NDP (neighbor discovery)
# locally instead of flooding to all VTEPs

# Arista EOS:
router bgp 65101
   address-family evpn
      neighbor SPINE activate
      route-type mac-ip advertise

! Configure NDP suppression on SVI
interface Vlan100
   ipv6 nd ra suppress all   ! Suppress RA flooding in VXLAN
   ! EVPN handles RA proxy instead

# FRR/Cumulus - NDP suppression
# bridge parameter neigh-suppress yes
ip link set vxlan100 type bridge_slave neigh_suppress on
bridge link show vxlan100
```

IPv6 VXLAN overlays provide L2 extension for IPv6 workloads across data center fabrics, with BGP EVPN Type-2 (MAC/IP) routes distributing IPv6 host bindings to all VTEPs, NDP suppression eliminating multicast flooding for neighbor discovery, and anycast gateway ensuring optimal L3 forwarding regardless of which leaf a workload is attached to.
