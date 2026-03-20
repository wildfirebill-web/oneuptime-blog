# How to Design IPv6 for Hyperscale Data Centers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Hyperscale, Data Center, BGP, ECMP, Clos

Description: Design IPv6 for hyperscale data center Clos network fabrics with BGP-only routing, ECMP load balancing, and large-scale address planning.

## Hyperscale IPv6 Architecture

Hyperscale data centers (Google, Facebook/Meta, Amazon) use Clos fabric topologies with BGP-only routing. Key IPv6 design principles:

- **BGP unnumbered**: uses link-local addresses (no IPv6 assignments per link)
- **/126 or /127 for P2P links**: conserves address space
- **/48 per pod**: hierarchical allocation for summarization
- **ECMP with 64+ paths**: load balance across all fabric paths
- **No IGP**: only iBGP between spine/leaf tiers

## Address Plan for Hyperscale

```
Hyperscale DC address plan:
Total DC allocation: 2001:db8::/32

Pod 1: 2001:db8:pod1::/48
  Leaf pair 1: 2001:db8:pod1:l1::/52
  Leaf pair 2: 2001:db8:pod1:l2::/52
  ...
  Server /64s: 2001:db8:pod1:l1:0000::/64 per rack

Spine tier: 2001:db8:spine::/48
  Management: 2001:db8:mgmt::/48

BGP loopbacks (for iBGP):
  Leaf-1:  2001:db8:loopback::1/128
  Leaf-2:  2001:db8:loopback::2/128
  Spine-1: 2001:db8:loopback::101/128
```

## BGP Unnumbered for Fabric Links

```bash
# FRR (Free Range Routing) — BGP unnumbered
# Uses link-local addresses for BGP peering (no IPv6 per link needed)

# /etc/frr/frr.conf on leaf switch

interface eth1
  ipv6 nd ra-interval 10
  no ipv6 nd suppress-ra
  # Link-local only — no global IPv6 address

router bgp 65001
  bgp router-id 1.1.1.1
  bgp bestpath as-path multipath-relax

  # BGP unnumbered peer
  neighbor eth1 interface remote-as external
  neighbor eth2 interface remote-as external
  neighbor eth3 interface remote-as external
  neighbor eth4 interface remote-as external

  address-family ipv6 unicast
    neighbor eth1 activate
    neighbor eth2 activate
    redistribute connected
    # Advertise server /64 subnets
    network 2001:db8:pod1:l1::/64
```

## ECMP Configuration

```bash
# Enable maximum ECMP paths in FRR

router bgp 65001
  address-family ipv6 unicast
    maximum-paths 64          # 64-way ECMP on spine
    maximum-paths ibgp 64

# Linux kernel ECMP
sysctl -w net.ipv6.fib_multipath_hash_policy=1  # 4-tuple hash
sysctl -w net.ipv6.fib_multipath_use_neigh=1

# Verify ECMP routes
ip -6 route show 2001:db8:pod2::/48 | head -10
# Should show multiple nexthops
```

## Server-Leaf Connectivity

```bash
# Server gets /64 via SLAAC from leaf router
# Leaf advertises /64 into BGP fabric

# On leaf router (radvd)
cat > /etc/radvd.conf << 'EOF'
interface eth-server-1 {
    AdvSendAdvert on;
    MinRtrAdvInterval 3;
    MaxRtrAdvInterval 10;
    AdvDefaultLifetime 30;

    prefix 2001:db8:pod1:l1:1::/80 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 300;
        AdvPreferredLifetime 120;
    };
};
EOF

# BGP: advertise server subnets from leaf
# (auto-advertised via "redistribute connected")
vtysh -c "show bgp ipv6 unicast 2001:db8:pod1:l1::/64"
```

## Anycast Gateway for Hyperscale

```bash
# All leaf switches share the same gateway IPv6 address
# Server's default gateway is always the same IP regardless of leaf

ANYCAST_GW="2001:db8:pod1:gw::1"
ANYCAST_MAC="02:00:00:00:00:01"  # Shared virtual MAC

# On each leaf:
ip link set dev br-servers address ${ANYCAST_MAC}
ip -6 addr add ${ANYCAST_GW}/64 dev br-servers

# Server configured with anycast gateway:
ip -6 route add default via ${ANYCAST_GW}
# Works regardless of which leaf the server connects to
```

## Scale Validation

```bash
#!/bin/bash
# validate-hyperscale.sh — Check routing table scale

echo "=== IPv6 Routing Table ==="
ip -6 route show | wc -l
echo "routes"

echo ""
echo "=== BGP Summary ==="
vtysh -c "show bgp ipv6 unicast summary" | tail -20

echo ""
echo "=== ECMP Paths ==="
# Count routes with multiple nexthops
vtysh -c "show bgp ipv6 unicast" | grep -c "=>"

echo ""
echo "=== NDP Cache ==="
ip -6 neigh show | wc -l
echo "NDP entries"
# Alert if approaching gc_thresh3
THRESH3=$(sysctl -n net.ipv6.neigh.default.gc_thresh3)
CURRENT=$(ip -6 neigh show | wc -l)
echo "NDP utilization: ${CURRENT}/${THRESH3} ($(( CURRENT * 100 / THRESH3 ))%)"
```

## Conclusion

Hyperscale IPv6 data center design centers on BGP-only Clos fabrics with BGP unnumbered peering (link-local addresses, no per-link IPv6 assignments). Allocate /48 per pod with /64 server subnets. Enable 64-way ECMP in FRR with `maximum-paths 64` and kernel multipath hash. Use anycast gateway — all leaf switches share one IPv6 gateway address and MAC — enabling servers to move between racks without reconfiguration. The NDP cache size is the critical constraint at scale — set `gc_thresh3` to 2× the number of servers per spine to prevent NDP table overflow.
