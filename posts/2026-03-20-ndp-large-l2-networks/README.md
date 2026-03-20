# How to Scale NDP in Large L2 IPv6 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, NDP, Scalability, L2, Data Center, Flooding

Description: Address NDP scalability challenges in large L2 IPv6 networks including ND cache overflow, flooding, and suppression techniques.

## NDP Scalability Problem

In large L2 domains (e.g., data center VLANs with thousands of VMs), NDP creates significant challenges:

- **NS flooding**: Neighbor Solicitations sent to multicast (solicited-node multicast or all-nodes) flood the entire L2 domain
- **Cache overflow**: Gateway devices need NDP cache entries for every host
- **ND storms**: Mass VM startup events trigger simultaneous DAD/NS flooding

A flat VLAN with 10,000 VMs generates significant NDP traffic:
- Each VM: ~1 NS/30s for reachability probing
- = 333 NS packets/second constantly
- Each NS hits all connected switches

## NDP Suppression on Linux Bridge/VXLAN

```bash
# Enable NDP suppression on a bridge (Linux 4.10+)
# Bridge learns addresses from NA/NS messages and responds locally

ip link add br100 type bridge
ip link set br100 type bridge neigh_suppress on
ip link set br100 type bridge vlan_filtering on

# Verify
bridge link show | grep neigh_suppress

# Add static NDP entries (pre-populate from CMDB)
ip -6 neigh add 2001:db8:100::10 \
    lladdr 52:54:00:11:22:33 \
    dev br100 nud permanent

ip -6 neigh add 2001:db8:100::11 \
    lladdr 52:54:00:44:55:66 \
    dev br100 nud permanent
```

## NDP Suppression on VXLAN Fabric

EVPN BGP distributes MAC/IP bindings and enables NDP suppression at VTEP level:

```bash
# VXLAN with NDP suppression
ip link add vxlan100 type vxlan \
    id 100 \
    local 2001:db8:1::1 \
    dstport 4789 \
    nolearning  # Disable data-plane learning (use BGP EVPN)

ip link add br100 type bridge
ip link set br100 type bridge neigh_suppress on
ip link set vxlan100 master br100

# BGP EVPN distributes Type 2 routes (MAC/IP)
# VTEP responds locally to NS without flooding
```

## FRR BGP EVPN with NDP Suppression

```
! FRRouting configuration for EVPN NDP suppression

router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 2001:db8:0:1::1 remote-as 65001
 !
 address-family l2vpn evpn
  neighbor 2001:db8:0:1::1 activate
  advertise-all-vni
 exit-address-family

! VNI/VXLAN interface
interface vxlan100
 ip address 2001:db8:1::1/64
 vxlan id 100
 vxlan local-tunnelip 2001:db8:1::1
 bridge-access 100
```

## Neighbor Cache Size Planning

```bash
#!/bin/bash
# Calculate required NDP cache size for a data center

TOTAL_VMS=10000
GATEWAY_VLANS=100
VMS_PER_VLAN=$((TOTAL_VMS / GATEWAY_VLANS))

echo "Data center NDP cache planning:"
echo "  Total VMs: ${TOTAL_VMS}"
echo "  VLANs: ${GATEWAY_VLANS}"
echo "  VMs per VLAN: ${VMS_PER_VLAN}"
echo ""
echo "Required cache sizes:"
echo "  Per-VLAN L3 gateway: ${VMS_PER_VLAN} entries"
echo "  Spine router (all VLANs): ${TOTAL_VMS} entries"
echo ""

# Set appropriate gc_thresh values
RECOMMENDED=$((TOTAL_VMS * 2))  # 2x for headroom
echo "Recommended gc_thresh3: ${RECOMMENDED}"

# Apply on spine routers
sysctl -w net.ipv6.neigh.default.gc_thresh3=${RECOMMENDED}
sysctl -w net.ipv6.neigh.default.gc_thresh2=$((RECOMMENDED / 2))
```

## Rate Limiting NDP at the Hypervisor

```bash
# Limit NDP NS rate per VM using nftables

# Allow NDP but rate-limit NS to prevent storms
nft add table ip6 ndp_ratelimit
nft add chain ip6 ndp_ratelimit forward '{ type filter hook forward priority 0; }'

# Rate limit Neighbor Solicitations: max 10/sec per VM
nft add rule ip6 ndp_ratelimit forward \
    icmpv6 type nd-neighbor-solicit \
    limit rate 10/second burst 20 packets \
    accept

nft add rule ip6 ndp_ratelimit forward \
    icmpv6 type nd-neighbor-solicit \
    drop
```

## Monitoring NDP Load

```bash
# Count NDP message rates
tcpdump -i eth0 -n 'icmp6 and (ip6[40]==135 or ip6[40]==136)' \
    -ttt -l 2>/dev/null | \
    awk '{ns++} NR%100==0 {print "Rate: " ns/10 " NDP/s"; ns=0}'

# Monitor cache entries per state
watch -n 5 "ip -6 neigh show | awk '{print \$5}' | sort | uniq -c"

# Check for NDP overflow (failed entries)
FAILED=$(ip -6 neigh show nud failed | wc -l)
echo "Failed NDP entries: ${FAILED}"
[ ${FAILED} -gt 100 ] && echo "WARNING: High NDP failure count"
```

## Conclusion

Large L2 IPv6 networks face NDP scalability challenges from flooding and cache overflow. The solutions are: NDP suppression on bridges (responds to NS locally from cache), BGP EVPN for control-plane MAC/IP distribution (eliminates data-plane flooding), adequate `gc_thresh3` for large neighbor tables, and rate limiting NS at hypervisor level. For data centers with 10,000+ VMs, NDP suppression is not optional — it's a requirement to prevent multicast flooding from degrading network performance.
