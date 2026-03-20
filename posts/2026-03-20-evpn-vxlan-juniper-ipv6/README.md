# How to Configure EVPN VXLAN with IPv6 on Juniper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Juniper, EVPN, VXLAN, IPv6, QFX, Data Center, BGP

Description: Configure BGP EVPN with VXLAN over an IPv6 underlay on Juniper QFX switches for scalable data center overlay networking.

## IPv6 Underlay Configuration (Junos)

```text
# Junos configuration on QFX Leaf

# Loopback for VTEP source
set interfaces lo0 unit 0 family inet6 address 2001:db8:1::1/128

# Fabric-facing interface
set interfaces xe-0/0/1 unit 0 family inet6 address 2001:db8:f:1::1/64

# OSPFv3 for IPv6 underlay reachability
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 area 0.0.0.0 interface xe-0/0/1.0
```

## BGP EVPN Configuration

```text
# BGP with IPv6 underlay - sessions use IPv6 loopbacks
set protocols bgp group EVPN-OVERLAY type internal
set protocols bgp group EVPN-OVERLAY local-address 2001:db8:1::1
set protocols bgp group EVPN-OVERLAY family evpn signaling
set protocols bgp group EVPN-OVERLAY neighbor 2001:db8:0:1::1
set protocols bgp group EVPN-OVERLAY neighbor 2001:db8:0:2::1

# Autonomous system
set routing-options autonomous-system 65001
```

## VXLAN Tunnel (VTEP) Definition

```text
# Define tunnel endpoint using IPv6 loopback
set interfaces vtep unit 0 tunnel-endpoint vxlan
set interfaces vtep unit 0 family inet6

# Bind VNI to tunnel
set vlans vlan100 vxlan vni 10100
set vlans vlan200 vxlan vni 10200

# EVPN for VXLAN
set protocols evpn vni-options vni 10100 vrf-target target:65001:10100
set protocols evpn vni-options vni 10200 vrf-target target:65001:10200
set protocols evpn encapsulation vxlan
set protocols evpn extended-vni-list all
set protocols evpn default-gateway no-gateway-community
```

## IRB for Distributed Anycast Gateway

```text
# Integrated Routing and Bridging (IRB) interface
set interfaces irb unit 100 family inet address 10.100.0.1/24 virtual-gateway-address 10.100.0.254
set interfaces irb unit 100 family inet6 address 2001:db8:100::1/64 virtual-gateway-address 2001:db8:100::ffff
set interfaces irb unit 100 mac 00:00:de:ad:be:ef  # Same MAC on all leafs

# Tenant routing instance
set routing-instances tenant1 instance-type vrf
set routing-instances tenant1 interface irb.100
set routing-instances tenant1 interface irb.200
set routing-instances tenant1 vrf-target target:65001:100
set routing-instances tenant1 protocols evpn ip-prefix-routes advertise direct-nexthop
```

## Route-Target and RD Policy

```python
# Import/export policy for EVPN routes
set policy-options policy-statement EVPN-EXPORT term 1 from protocol direct
set policy-options policy-statement EVPN-EXPORT term 1 then community add RT65001
set policy-options policy-statement EVPN-EXPORT term 1 then accept

set policy-options community RT65001 members target:65001:100

set routing-instances tenant1 vrf-import EVPN-IMPORT
set routing-instances tenant1 vrf-export EVPN-EXPORT
```

## Verification Commands

```text
# Show BGP EVPN peers
show bgp summary | match EVPN

# Show EVPN database (MAC/IP routes)
show evpn database
show evpn database extensive

# Show VXLAN tunnel information
show evpn instance
show evpn encapsulation statistics

# Show MAC table
show ethernet-switching table

# Show ARP/NDP in VRF
show arp routing-instance tenant1
show ipv6 neighbors routing-instance tenant1

# Ping over overlay
ping routing-instance tenant1 10.100.0.2
ping routing-instance tenant1 2001:db8:100::2
```

## Spine Route Reflector Configuration

```text
# Junos spine as BGP Route Reflector
set protocols bgp group LEAF-OVERLAY type internal
set protocols bgp group LEAF-OVERLAY cluster 10.0.0.1
set protocols bgp group LEAF-OVERLAY family evpn signaling
set protocols bgp group LEAF-OVERLAY neighbor 2001:db8:1::1 description Leaf-1
set protocols bgp group LEAF-OVERLAY neighbor 2001:db8:2::1 description Leaf-2
set protocols bgp group LEAF-OVERLAY neighbor 2001:db8:3::1 description Leaf-3
```

## Conclusion

Juniper QFX EVPN VXLAN with IPv6 underlay uses OSPFv3 for underlay reachability and BGP sessions between IPv6 loopbacks. The `protocols evpn` stanza with `encapsulation vxlan` configures the data plane. IRB interfaces with virtual gateway addresses implement the distributed anycast gateway for optimal L3 forwarding. Route targets control tenant isolation across the fabric. Spine nodes act as BGP route reflectors, scaling the control plane without requiring full-mesh BGP between all leaf nodes.
