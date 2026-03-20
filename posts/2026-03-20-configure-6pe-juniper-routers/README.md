# How to Configure 6PE on Juniper Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 6PE, Juniper, JunOS, MPLS, BGP, LDP

Description: Configure 6PE (IPv6 Provider Edge) on Juniper routers running JunOS, including MPLS LDP configuration, MP-BGP inet6 address family, and verification commands.

---

Juniper JunOS supports 6PE through the `inet6` address family in BGP with labeled-unicast configuration. PE routers advertise IPv6 prefixes as labeled routes via MP-BGP, allowing IPv6 transit across an IPv4-only MPLS backbone.

## JunOS MPLS and LDP Configuration

```
# PE1 JunOS - Configure MPLS and LDP on backbone interfaces
set protocols mpls interface ge-0/0/0.0
set protocols ldp interface ge-0/0/0.0

# Configure OSPF for IPv4 backbone routing
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface lo0.0

# Enable MPLS on backbone interface
set interfaces ge-0/0/0 unit 0 family mpls

# Verify MPLS is configured
show interfaces ge-0/0/0 | match mpls
```

## JunOS BGP IPv6 for 6PE

```
# PE1 - Configure BGP group for PE-PE 6PE sessions
set protocols bgp group PE-iBGP type internal
set protocols bgp group PE-iBGP local-address 10.0.0.1
set protocols bgp group PE-iBGP family inet6 labeled-unicast
set protocols bgp group PE-iBGP neighbor 10.0.0.2

# For 6PE, use inet6 labeled-unicast (not standard inet6)
# This causes BGP to attach MPLS labels to IPv6 prefixes

# CE-PE peering (eBGP, IPv6)
set protocols bgp group CE-IPv6 type external
set protocols bgp group CE-IPv6 family inet6 unicast
set protocols bgp group CE-IPv6 peer-as 65001
set protocols bgp group CE-IPv6 neighbor 2001:db8:pe1-ce1::2

# Configure local PE IPv6 address for CE peering
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:pe1-ce1::1/64
```

## JunOS PE Router Full Configuration

```
# /etc/junos (equivalent set commands)

# Interfaces
set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set interfaces lo0 unit 0 family inet6 address 2001:db8:pe1::1/128

set interfaces ge-0/0/0 unit 0 description "To MPLS Core"
set interfaces ge-0/0/0 unit 0 family inet address 10.1.1.1/30
set interfaces ge-0/0/0 unit 0 family mpls

set interfaces ge-0/0/1 unit 0 description "To CE1 IPv6"
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:pe1-ce1::1/64

# MPLS
set protocols mpls interface ge-0/0/0.0
set protocols mpls interface lo0.0

# LDP
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface lo0.0

# OSPF (backbone IGP)
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface lo0.0 passive

# BGP
set protocols bgp group IBGP type internal
set protocols bgp group IBGP local-address 10.0.0.1
set protocols bgp group IBGP family inet6 labeled-unicast
set protocols bgp group IBGP neighbor 10.0.0.2

set protocols bgp group CE-PEERING type external
set protocols bgp group CE-PEERING peer-as 65001
set protocols bgp group CE-PEERING family inet6 unicast
set protocols bgp group CE-PEERING neighbor 2001:db8:pe1-ce1::2
```

## CE Router JunOS Configuration

```
# Customer Edge Router
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:pe1-ce1::2/64

# Add customer prefix route for BGP advertisement
set routing-options rib inet6.0 static route 2001:db8:site1::/48 discard

# BGP to PE
set protocols bgp group PE-PEERING type external
set protocols bgp group PE-PEERING peer-as 65000
set protocols bgp group PE-PEERING family inet6 unicast
set protocols bgp group PE-PEERING neighbor 2001:db8:pe1-ce1::1 local-address 2001:db8:pe1-ce1::2
set protocols bgp group PE-PEERING export EXPORT-SITE1

# Export policy: advertise customer prefix
set policy-options policy-statement EXPORT-SITE1 term 1 from protocol static
set policy-options policy-statement EXPORT-SITE1 term 1 from route-filter 2001:db8:site1::/48 exact
set policy-options policy-statement EXPORT-SITE1 term 1 then accept
set policy-options policy-statement EXPORT-SITE1 term default then reject
```

## JunOS 6PE Verification

```bash
# Check BGP sessions
show bgp summary
# Should show: PE2 iBGP established, CE1 eBGP established

# View IPv6 BGP table with labels
show route table inet6.0 detail | grep -A5 "2001:db8:site2"
# Look for: Label-based Forwarding: Yes

# Alternatively:
show bgp neighbor 10.0.0.2 received-routes
# Shows IPv6 routes received from PE2

# Check MPLS labeled routes
show route table inet6.3
# JunOS uses inet6.3 for IPv6 labeled routes

# Verify forwarding
show route table inet6.0 forwarding-type unicast

# Check MPLS forwarding
show mpls label-switched-path detail
show route forwarding-table family inet6

# Test end-to-end
ping inet6 2001:db8:site2::10 source 2001:db8:site1::1
traceroute inet6 2001:db8:site2::10 source 2001:db8:site1::1
```

## Policy for 6PE Next-Hop

```
# JunOS - BGP export policy for 6PE
# Set IPv4-mapped IPv6 next-hop for iBGP advertisement

set policy-options policy-statement 6PE-EXPORT term 1 \
    from protocol bgp
set policy-options policy-statement 6PE-EXPORT term 1 \
    from family inet6
set policy-options policy-statement 6PE-EXPORT term 1 \
    then accept

set protocols bgp group IBGP export 6PE-EXPORT

# Verify next-hop format (should be IPv4-mapped IPv6: ::ffff:10.0.0.x)
show bgp neighbor 10.0.0.2 advertised-routes | grep next-hop
```

Juniper JunOS 6PE requires `family inet6 labeled-unicast` in the iBGP group configuration to enable MPLS label attachment to IPv6 prefixes, with JunOS automatically managing the IPv4-mapped IPv6 next-hop format needed for MPLS label resolution, and routes appearing in the `inet6.3` table for labeled IPv6 forwarding.
