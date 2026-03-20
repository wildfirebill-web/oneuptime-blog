# How to Configure IPv6 for Data Center Interconnect (DCI)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DCI, Data Center, DWDM, Dark Fiber, Networking

Description: Configure IPv6 routing and tunneling for data center interconnect links including BGP, OSPF, and EVPN for multi-site IPv6 deployments.

## DCI IPv6 Design Options

| Method | Use Case | Protocol |
|---|---|---|
| Native IPv6 routed | Simple site-to-site | OSPFv3, BGP |
| EVPN-VXLAN | L2 extension across sites | BGP EVPN |
| L3VPN with IPv6 | Multi-tenant DCI | MP-BGP VPNv6 |
| IPv6 over MPLS | Service provider DCI | LDP/RSVP |

## Native IPv6 DCI with BGP

```
! Cisco NX-OS — DCI link between DC1 and DC2

! DC1 (2001:db8:dc1::/48) interconnect interface
interface Ethernet1/1
  description DCI_to_DC2
  ipv6 address 2001:db8:transit::1/64
  ipv6 ospfv3 1 area 0

router ospfv3 1
  router-id 1.1.1.1
  address-family ipv6 unicast
    redistribute bgp 65001 route-map EXPORT_TO_OSPF

router bgp 65001
  router-id 1.1.1.1
  neighbor 2001:db8:dc2::router remote-as 65002
  address-family ipv6 unicast
    neighbor 2001:db8:dc2::router activate
    network 2001:db8:dc1::/48
```

## EVPN-VXLAN DCI for L2 Extension

```bash
# Linux VTEP with EVPN for DCI L2 extension

# Create VXLAN interface at DC1
ip link add vxlan100 type vxlan \
    id 100 \
    local 2001:db8:dc1::vtep \
    dstport 4789 \
    nolearning

ip link add br100 type bridge
ip link set br100 type bridge neigh_suppress on
ip link set vxlan100 master br100
ip link set vxlan100 up
ip link set br100 up

# FRR BGP EVPN for DCI
# /etc/frr/frr.conf
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 2001:db8:dc2::router remote-as 65002
 address-family l2vpn evpn
  neighbor 2001:db8:dc2::router activate
  advertise-all-vni
```

## Path MTU for DCI

```bash
# DCI links often have different MTU than intra-DC
# VXLAN over IPv6 = 50 bytes overhead

# DC1 fabric MTU: 9216
# DCI link MTU: 9000 (common for 100G DCI)
# VXLAN overhead: 50 bytes
# Effective payload: 8950 bytes

# Set MTU on DCI interface
ip link set dev eth-dci mtu 9000

# Enable PMTU discovery for DCI traffic
sysctl -w net.ipv6.conf.eth-dci.mtu_expires=600

# Test DCI path MTU
ping6 -M do -s 8900 2001:db8:dc2::router
```

## BGP Communities for DCI Traffic Engineering

```
# Tag routes with DCI-specific communities

route-map DCI_EXPORT permit 10
  set community 65001:1000  # DCI route tag
  set local-preference 150  # Prefer direct path over DCI

# DC2 BGP: apply MED for DCI path selection
router bgp 65002
  neighbor 2001:db8:dc1::router route-map DCI_IMPORT in

route-map DCI_IMPORT permit 10
  match community DCI_ROUTES
  set metric 100  # Higher metric = lower preference
```

## Monitoring DCI Links

```bash
#!/bin/bash
# monitor-dci.sh — DCI IPv6 health check

DC1_ROUTER="2001:db8:dc1::router"
DC2_ROUTER="2001:db8:dc2::router"

echo "=== DCI Link Health ==="

# Ping test
if ping6 -c 3 -W 2 "${DC2_ROUTER}" &>/dev/null; then
    echo "PASS: DCI link to DC2 is up"
else
    echo "FAIL: DCI link to DC2 is down"
fi

# BGP session check
if vtysh -c "show bgp summary" | grep "${DC2_ROUTER}" | grep -q "Establ"; then
    echo "PASS: BGP session to DC2 is established"
else
    echo "FAIL: BGP session to DC2 is down"
fi

# Route check
ROUTES=$(ip -6 route show | grep "2001:db8:dc2::" | wc -l)
echo "IPv6 routes to DC2: ${ROUTES}"
```

## Conclusion

IPv6 DCI configuration depends on the L2/L3 requirements. For pure L3 routing, use OSPFv3 or eBGP between sites with site-specific /48 prefixes advertised across the DCI link. For L2 extension (VM migration, stretched clusters), use EVPN-VXLAN with BGP EVPN Type 2 MAC/IP routes propagated between sites. Always account for DCI MTU — VXLAN over IPv6 adds 50 bytes overhead, so ensure the DCI link supports at least 9050 bytes for jumbo frame workloads. Use BGP communities to tag DCI-learned routes for traffic engineering and prefer intra-DC paths for local traffic.
