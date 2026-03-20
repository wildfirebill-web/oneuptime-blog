# How to Configure 6VPE on Juniper Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6VPE, Juniper, JunOS, MPLS, BGP, VRF, L3VPN, Routing Instance

Description: Configure 6VPE (IPv6 VPN Provider Edge) on Juniper JunOS routers using routing instances, VPNv6 address family in MP-BGP, and route distinguisher/target policies for enterprise IPv6 VPN services.

---

Juniper JunOS implements 6VPE using routing instances (similar to Cisco VRFs) with the `vrf` type and `inet6` protocol family. MP-BGP uses the `inet6-vpn unicast` address family to exchange VPNv6 routes with RD/RT extended communities between PE routers.

## JunOS Routing Instance (VRF) for 6VPE

```
# Create routing instance for Customer A IPv6 VPN
set routing-instances CUSTOMER-A instance-type vrf
set routing-instances CUSTOMER-A route-distinguisher 65000:100
set routing-instances CUSTOMER-A vrf-target target:65000:100
set routing-instances CUSTOMER-A protocols bgp group CE-A family inet6 unicast

# Add CE-facing interface to routing instance
set routing-instances CUSTOMER-A interface ge-0/0/1.0

# Configure interface
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:pe1-cea::1/64

# Customer B
set routing-instances CUSTOMER-B instance-type vrf
set routing-instances CUSTOMER-B route-distinguisher 65000:200
set routing-instances CUSTOMER-B vrf-target target:65000:200
set routing-instances CUSTOMER-B interface ge-0/0/2.0
```

## JunOS MP-BGP VPNv6 Address Family

```
# PE1 - Enable VPNv6 (inet6-vpn) in iBGP group
set protocols bgp group PE-IBGP type internal
set protocols bgp group PE-IBGP local-address 10.0.0.1

# inet6-vpn unicast = VPNv6 address family
set protocols bgp group PE-IBGP family inet6-vpn unicast

# iBGP neighbor (PE2)
set protocols bgp group PE-IBGP neighbor 10.0.0.2

# CE-facing BGP in routing instance (per-VRF peering)
set routing-instances CUSTOMER-A protocols bgp group CE-A-BGP type external
set routing-instances CUSTOMER-A protocols bgp group CE-A-BGP peer-as 65001
set routing-instances CUSTOMER-A protocols bgp group CE-A-BGP family inet6 unicast
set routing-instances CUSTOMER-A protocols bgp group CE-A-BGP neighbor 2001:db8:pe1-cea::2
set routing-instances CUSTOMER-A protocols bgp group CE-A-BGP export EXPORT-CE-ROUTES
```

## Complete PE1 6VPE Configuration

```
# PE1 Full Configuration

# MPLS backbone
set interfaces ge-0/0/0 unit 0 description "MPLS Core"
set interfaces ge-0/0/0 unit 0 family inet address 10.1.1.1/30
set interfaces ge-0/0/0 unit 0 family mpls

# OSPF + LDP for backbone
set protocols ospf area 0 interface ge-0/0/0.0
set protocols ospf area 0 interface lo0.0 passive
set protocols ldp interface ge-0/0/0.0
set protocols mpls interface ge-0/0/0.0

# Loopback
set interfaces lo0 unit 0 family inet address 10.0.0.1/32

# Global BGP with VPNv6 AF
set protocols bgp group IBGP type internal
set protocols bgp group IBGP local-address 10.0.0.1
set protocols bgp group IBGP family inet-vpn unicast
set protocols bgp group IBGP family inet6-vpn unicast
set protocols bgp group IBGP neighbor 10.0.0.2

# Customer A VRF
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:pe1-cea::1/64
set routing-instances CUSTOMER-A instance-type vrf
set routing-instances CUSTOMER-A route-distinguisher 65000:100
set routing-instances CUSTOMER-A vrf-target target:65000:100
set routing-instances CUSTOMER-A interface ge-0/0/1.0
set routing-instances CUSTOMER-A protocols bgp group CE type external
set routing-instances CUSTOMER-A protocols bgp group CE peer-as 65001
set routing-instances CUSTOMER-A protocols bgp group CE family inet6 unicast
set routing-instances CUSTOMER-A protocols bgp group CE neighbor 2001:db8:pe1-cea::2

# Customer B VRF
set interfaces ge-0/0/2 unit 0 family inet6 address 2001:db8:pe1-ceb::1/64
set routing-instances CUSTOMER-B instance-type vrf
set routing-instances CUSTOMER-B route-distinguisher 65000:200
set routing-instances CUSTOMER-B vrf-target target:65000:200
set routing-instances CUSTOMER-B interface ge-0/0/2.0
set routing-instances CUSTOMER-B protocols bgp group CE type external
set routing-instances CUSTOMER-B protocols bgp group CE peer-as 65002
set routing-instances CUSTOMER-B protocols bgp group CE family inet6 unicast
set routing-instances CUSTOMER-B protocols bgp group CE neighbor 2001:db8:pe1-ceb::2
```

## JunOS 6VPE Verification

```bash
# Show VPNv6 BGP summary
show bgp summary | grep "inet6-vpn\|Establ"

# View VPNv6 routes in BGP
show route table bgp.l3vpn-inet6.0
# Shows: RD:prefix, label, RT attributes

# Check per-routing-instance IPv6 routes
show route table CUSTOMER-A.inet6.0
# Should show remote site IPv6 prefixes

# Verify VPN labels
show route table CUSTOMER-A.inet6.0 detail | grep "Label\|Nexthop\|Communities"

# Check MPLS forwarding for VPN
show route forwarding-table family inet6 table CUSTOMER-A

# Test connectivity
ping routing-instance CUSTOMER-A inet6 2001:db8:cust-a-site2::10

# Traceroute through VPN
traceroute routing-instance CUSTOMER-A inet6 2001:db8:cust-a-site2::10

# Verify isolation (Customer A cannot reach B)
ping routing-instance CUSTOMER-A inet6 2001:db8:cust-b-site1::10
# Should fail: No route to host
```

## Hub-and-Spoke 6VPE (Shared Services)

```
# Hub site (data center) with shared services reachable by all VPNs
# Use extra RT for spoke import

# Hub VRF: exports with both specific and shared RT
set routing-instances HUB-SITE vrf-export HUB-EXPORT
set policy-options policy-statement HUB-EXPORT term 1 then community add HUB-RT
set policy-options policy-statement HUB-EXPORT term 1 then community add SPOKE-RT
set policy-options policy-statement HUB-EXPORT term 1 then accept

# Spoke VRFs: import hub's shared RT
set routing-instances CUSTOMER-A vrf-import SPOKE-IMPORT
set policy-options policy-statement SPOKE-IMPORT term import-hub \
    from community HUB-RT
set policy-options policy-statement SPOKE-IMPORT term import-hub then accept
set policy-options policy-statement SPOKE-IMPORT term import-own \
    from community SPOKE-RT
set policy-options policy-statement SPOKE-IMPORT term import-own then accept
```

JunOS 6VPE uses routing instances of type `vrf` with `inet6` address family, `inet6-vpn unicast` in the iBGP group for VPNv6 route exchange, and `vrf-target` as a simple way to configure matching RD/RT values, resulting in per-customer IPv6 routing isolation with efficient two-label MPLS forwarding across the service provider backbone.
