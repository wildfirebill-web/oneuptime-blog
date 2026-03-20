# How to Configure 6VPE on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6VPE, Cisco, IOS, MPLS, BGP, VRF, L3VPN

Description: Configure 6VPE (IPv6 VPN Provider Edge) on Cisco IOS and IOS XE routers for delivering IPv6 L3VPN services, including VRF configuration, VPNv6 address family setup, and CE-PE peering.

---

6VPE adds VRF-based customer isolation to 6PE, enabling multiple enterprise customers to use separate IPv6 VPNs over the same MPLS backbone. Configuration requires VRF definitions with RD/RT for IPv6, VPNv6 address family in MP-BGP, and CE-facing IPv6 interfaces associated with VRFs.

## 6VPE Configuration on PE Router

```bash
! PE1 - VRF Definition for Customer A (IPv6 VPN)
vrf definition CUSTOMER-A
 rd 65000:100
 !
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family
 !
 address-family ipv6
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family

! VRF for Customer B
vrf definition CUSTOMER-B
 rd 65000:200
 !
 address-family ipv6
  route-target export 65000:200
  route-target import 65000:200
 exit-address-family
```

## CE-PE Interface Configuration

```text
! PE1 - CE-facing interface for Customer A
interface GigabitEthernet0/1
 description Customer-A-CE-Site1
 vrf forwarding CUSTOMER-A
 ipv6 address 2001:db8:pe1-cea::1/64
 ipv6 enable
 no shutdown

! PE1 - CE-facing interface for Customer B
interface GigabitEthernet0/2
 description Customer-B-CE-Site1
 vrf forwarding CUSTOMER-B
 ipv6 address 2001:db8:pe1-ceb::1/64
 ipv6 enable
 no shutdown
```

## MP-BGP VPNv6 Address Family

```text
! PE1 - MP-BGP for 6VPE
router bgp 65000
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast

 ! iBGP neighbor (PE2)
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 neighbor 10.0.0.2 next-hop-self

 ! VPNv6 address family (for 6VPE)
 address-family vpnv6
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community extended
 exit-address-family

 ! Customer A VRF BGP - eBGP with CE1
 address-family ipv6 vrf CUSTOMER-A
  neighbor 2001:db8:pe1-cea::2 remote-as 65001
  neighbor 2001:db8:pe1-cea::2 activate
  neighbor 2001:db8:pe1-cea::2 description CE-A-Site1
  redistribute connected
 exit-address-family

 ! Customer B VRF BGP
 address-family ipv6 vrf CUSTOMER-B
  neighbor 2001:db8:pe1-ceb::2 remote-as 65002
  neighbor 2001:db8:pe1-ceb::2 activate
  redistribute connected
 exit-address-family
```

## CE Router Configuration

```text
! Customer A CE1 Router
interface GigabitEthernet0/0
 description To PE1
 ipv6 address 2001:db8:pe1-cea::2/64

! Add static summary route for BGP advertisement
ipv6 route 2001:db8:cust-a-site1::/48 Null0

! CE BGP to PE
router bgp 65001
 no bgp default ipv4-unicast

 address-family ipv6
  neighbor 2001:db8:pe1-cea::1 remote-as 65000
  neighbor 2001:db8:pe1-cea::1 activate
  neighbor 2001:db8:pe1-cea::1 description PE1
  network 2001:db8:cust-a-site1::/48
 exit-address-family
```

## Verify 6VPE Operation

```text
! Check VPNv6 BGP table
show bgp vpnv6 unicast all summary
! Should show PE2 iBGP peer established

! View VPNv6 routes (all VRFs)
show bgp vpnv6 unicast all
! Shows: RD:prefix, labels, RT

! View VPNv6 routes for specific customer
show bgp vpnv6 unicast rd 65000:100

! Check VRF-specific IPv6 routing table
show ipv6 route vrf CUSTOMER-A
! Should show: customer site2 prefix via PE2

! Check MPLS VPN label forwarding
show mpls forwarding-table vrf CUSTOMER-A
! Shows: incoming label, outgoing label, interface

! Test connectivity (Customer A VPN)
ping vrf CUSTOMER-A ipv6 2001:db8:cust-a-site2::10

! Traceroute through VPN
traceroute vrf CUSTOMER-A ipv6 2001:db8:cust-a-site2::10

! Verify customer isolation (Customer A cannot reach B)
ping vrf CUSTOMER-A ipv6 2001:db8:cust-b-site1::10
! Should fail (different VRF = isolated)
```

## Route Distinguisher Best Practices

```text
RD Allocation Strategies:

Strategy 1: AS:site-id
  RD: 65000:100  (ASN 65000, site 100)
  RD: 65000:101  (ASN 65000, site 101)
  Advantage: Simple, easy tracking

Strategy 2: IP:VPN-id
  RD: 10.0.0.1:100  (PE1 IP, VPN 100)
  Advantage: Unique even if same VPN across multiple PEs
  Use when PE maintains VRF state per-site

Strategy 3: IP:0 with different RTs for communities
  RD: 10.0.0.1:0
  RT export: 65000:100 65000:shared (multiple RTs)
  Use for hub-and-spoke or shared services VPNs
```

6VPE on Cisco requires VRF definitions with both IPv4 and IPv6 address families, VPNv6 address family in MP-BGP for inter-PE route exchange using extended communities, and per-VRF BGP peering with customer CEs, resulting in complete IPv6 routing isolation between customers while sharing the same MPLS backbone infrastructure.
