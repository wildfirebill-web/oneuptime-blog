# How to Understand SRv6 in Data Center Fabrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, Data Center, BGP, EVPN, Fabric, Networking

Description: Understand how SRv6 is deployed in data center Clos fabrics to replace MPLS-based overlays, simplify EVPN control planes, and enable traffic engineering.

## Introduction

Data centers use Clos (spine-leaf) fabrics where traffic engineering and multi-tenancy have traditionally required MPLS+EVPN. SRv6 replaces the MPLS data plane while retaining EVPN as the control plane, simplifying the architecture and eliminating MPLS label management overhead.

## SRv6 in a Clos Fabric

```
                    +------------------+
                    |   Spine Layer    |
                    |  5f00:spine::/48 |
                    +------------------+
                   /                    \
      +-----------+                      +-----------+
      |  Leaf 1   |                      |  Leaf 2   |
      | 5f00:1::/48|                    | 5f00:2::/48|
      +-----------+                      +-----------+
           |                                   |
      Server A (VRF Red)               Server B (VRF Red)
      fd00:red::a/64                   fd00:red::b/64
```

## BGP EVPN with SRv6 Transport

```
! Leaf 1 BGP configuration (FRR example)
router bgp 65001
  bgp router-id 1.1.1.1

  neighbor 5f00:spine:: remote-as 65000
  neighbor 5f00:spine:: update-source lo

  address-family l2vpn evpn
    neighbor 5f00:spine:: activate
    advertise-all-vni
    advertise-svi-ip
  !

  address-family ipv6 unicast
    network 5f00:1::/48  ! Advertise own locator
  !
!

! VRF with SRv6 End.DT6 SID
vrf RED
  vni 10100
  !

! Assign End.DT6 SID for VRF Red
! SID: 5f00:1:0:e100:: (function e100 = VRF Red)
segment-routing
  srv6
    locator MAIN
    !
  !
!
```

## SRv6 EVPN Control Plane Messages

```
EVPN Type-2 (MAC/IP route) with SRv6:
  Route: MAC=aa:bb:cc:dd:ee:ff, IP=fd00:red::a
  SRv6 L3 VPN SID: 5f00:1:0:e100::
  → Leaf 2 installs: route to fd00:red::a
    via encap seg6 mode encap segs [5f00:1:0:e100::]

EVPN Type-5 (IP prefix route) with SRv6:
  Prefix: fd00:red::/64
  SRv6 L3 VPN SID: 5f00:1:0:e100::
  → Remote leafs install forwarding entry with SRv6 encap
```

## Traffic Engineering in DC Fabric

```bash
# On Leaf 1: steer tenant traffic through specific spine
# Normal path: Leaf1 → Spine1 → Leaf2
# Engineered path: Leaf1 → Spine2 → Leaf2 (lower latency spine)

ip -6 route add 5f00:2::/48 \
  encap seg6 mode encap \
  segs 5f00:spine2:0:e001::,5f00:2:0:e000:: \
  dev eth-spine2

# This ensures VRF Red traffic from Leaf1 to Leaf2 uses Spine2
```

## ECMP and Load Balancing with SRv6

```bash
# SRv6 works with ECMP — multiple paths to same locator
# Leaf 1 has two spines: traffic is hashed across both

ip -6 route add 5f00:2::/48 \
  nexthop via fd00:l1-s1::spine1 dev eth0 weight 1 \
  nexthop via fd00:l1-s2::spine2 dev eth1 weight 1

# With SRv6, the outer IPv6 header provides 5-tuple for ECMP hashing:
# src=5f00:1::, dst=5f00:2:0:e100:: — deterministic per-flow

# Verify ECMP distribution
ethtool -S eth0 | grep rx_queue
```

## Monitoring SRv6 Fabric Health

```bash
# Ping all leaf locators from a spine
for leaf in 5f00:1:: 5f00:2:: 5f00:3:: 5f00:4::; do
  result=$(ping6 -c 2 -W 1 "$leaf" 2>&1 | grep -oP '\d+\.\d+ ms' | tail -1)
  echo "Leaf $leaf: $result"
done

# Monitor SRv6 encap counter (kernel stats)
ip -s -6 route show 5f00:2::/48 | grep -A3 "encap"
```

## Conclusion

SRv6 simplifies data center fabrics by combining the EVPN control plane with a pure IPv6 data plane. Each leaf's locator is a BGP prefix; EVPN routes carry SRv6 SIDs instead of MPLS labels. This eliminates label distribution protocols while retaining traffic engineering capabilities. Monitor per-leaf SID reachability and EVPN session health with OneUptime.
