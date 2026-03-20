# How to Understand 6VPE: IPv6 VPN over MPLS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6VPE, MPLS, BGP, VRF, L3VPN, VPN, Provider Edge

Description: Understand 6VPE (IPv6 VPN Provider Edge) architecture for delivering IPv6 L3VPN services over MPLS backbones, including VRF-based customer isolation and two-label MPLS forwarding.

---

6VPE (RFC 4659) extends BGP/MPLS L3VPN to support IPv6. Each VPN customer gets dedicated IPv6 VRFs on PE routers, with MP-BGP using the VPNv6 address family to exchange customer IPv6 prefixes with VPN labels. The backbone remains IPv4/MPLS with two-label stacks.

## 6VPE Architecture

```
6VPE Network Architecture:

Customer A:                          Customer A:
IPv6 Site 1 ─── [PE1/VRF-A] ═══ MPLS ═══ [PE2/VRF-A] ─── IPv6 Site 2
                     ║                         ║
Customer B:          ║  Two-label MPLS         ║         Customer B:
IPv6 Site 1 ─── [PE1/VRF-B]               [PE2/VRF-B] ─── IPv6 Site 2

MPLS Label Stack for 6VPE:
[LDP Transport Label | VPN Label] + IPv6 Packet
       ^                   ^
  Label for PE2      Label for VRF/customer
  (from LDP/RSVP)    (from MP-BGP VPNv6)

Customer isolation: VRFs ensure customers see only their own IPv6 routes
```

## VPNv6 Address Family

```
VPNv6 Route Distinguisher (RD) and Route Target (RT):

Each VPN customer IPv6 prefix has:
1. Route Distinguisher (RD): Makes prefix globally unique in BGP
   Format: ASN:NN or IP:NN (e.g., 65000:100 for Customer A)

2. Route Target (RT): Controls import/export between VRFs
   Import RT: Import routes with this RT into this VRF
   Export RT: Tag routes exported from this VRF with this RT

Example:
VRF Customer-A on PE1:
  RD: 65000:100
  Export RT: 65000:100
  Import RT: 65000:100

VRF Customer-A on PE2:
  RD: 65000:100        (same RD for same customer)
  Export RT: 65000:100
  Import RT: 65000:100

BGP VPNv6 NLRI: RD:IPv6prefix
  65000:100:2001:db8:site1::/48 (Site 1 prefix with RD)
  Label: 3000 (VPN label for customer A)
```

## Cisco IOS 6VPE Configuration

```
! PE Router - 6VPE Configuration

! IPv4/IPv6 dual-stack loopback for BGP
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
 ipv6 address 2001:db8:pe1::1/128

! MPLS on backbone interfaces
interface GigabitEthernet0/0
 description To MPLS Core
 ip address 10.1.1.1 255.255.255.252
 mpls ip

! VRF for Customer A IPv6 VPN
vrf definition CUSTOMER-A
 rd 65000:100
 address-family ipv6
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family

! CE-facing interface for Customer A
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER-A
 ipv6 address 2001:db8:pe1-ce-a::1/64
 ipv6 enable

! MP-BGP configuration
router bgp 65000
 !
 ! iBGP peer with PE2
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 !
 ! VPNv6 address family
 address-family vpnv6
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community extended
 exit-address-family

! BGP peering with Customer A CE
router bgp 65000
 address-family ipv6 vrf CUSTOMER-A
  neighbor 2001:db8:pe1-ce-a::2 remote-as 65001
  neighbor 2001:db8:pe1-ce-a::2 activate
  redistribute connected
 exit-address-family
```

## Verify 6VPE Operation

```bash
# Cisco IOS verification commands:

# Check VPNv6 routes in BGP
show bgp vpnv6 unicast all summary
show bgp vpnv6 unicast all

# View VRF-specific IPv6 routes
show ipv6 route vrf CUSTOMER-A

# Check MPLS forwarding for VPN labels
show mpls forwarding-table vrf CUSTOMER-A

# Verify VPNv6 label bindings
show bgp vpnv6 unicast all labels
# Example output:
# 2001:db8:site2::/48
#   10.0.0.2/32  VPN Label: 3001, Transport: 16

# Test connectivity through VPN
ping vrf CUSTOMER-A ipv6 2001:db8:site2::10 source 2001:db8:site1::1

# Traceroute through VPN
traceroute vrf CUSTOMER-A ipv6 2001:db8:site2::10
# Should show: MPLS labels in trace output
```

## 6VPE vs Standard IPv6 VRF

```
Key Differences:

Standard IPv6 VRF (no MPLS):
- Routes exchanged via ISIS/OSPF/static between sites
- Requires L3 connectivity between all PEs
- No label-based switching

6VPE (BGP/MPLS L3VPN for IPv6):
- Routes exchanged via MP-BGP VPNv6 address family
- Requires only IP connectivity between PEs (MPLS handles forwarding)
- Label stack: [transport-label | vpn-label] + IPv6
- Scales to thousands of VPN customers
- Complete customer isolation via VRFs
```

6VPE delivers BGP/MPLS L3VPN services for IPv6 customers by maintaining separate IPv6 VRFs per customer on PE routers, using the VPNv6 address family in MP-BGP to distribute customer IPv6 prefixes with two-level MPLS labels for transport and VPN identification, enabling ISPs to offer enterprise IPv6 VPN services over existing IPv4 MPLS infrastructure.
