# How to Configure IPv6 Policy Routing in SD-WAN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Policy Routing, SD-WAN, PBR, Route Maps, Traffic Engineering

Description: Configure IPv6 policy-based routing in SD-WAN environments to steer specific IPv6 traffic flows based on source/destination prefix, DSCP, application, or other criteria.

---

IPv6 policy-based routing (PBR) in SD-WAN directs IPv6 traffic based on policies rather than destination-only routing. This enables steering VoIP over low-latency links, video over high-bandwidth paths, and critical applications over premium MPLS while bulk traffic uses cheaper broadband.

## Linux IPv6 Policy Routing

```bash
#!/bin/bash
# ipv6-policy-routing.sh - Policy routing for SD-WAN IPv6

# Create routing tables for different WAN paths

# Table 200: MPLS (low latency, premium)
# Table 201: Broadband (high bandwidth, cheaper)
# Table 202: LTE (backup)

# Add routes to each table
# MPLS next-hop
ip -6 route add default via 2001:db8:mpls::gateway dev eth0 table 200
# Broadband next-hop
ip -6 route add default via 2001:db8:bb::gateway dev eth1 table 201
# LTE backup next-hop
ip -6 route add default via 2001:db8:lte::gateway dev eth2 table 202

# Create IPv6 rules for policy routing

# Rule 1: VoIP traffic (DSCP EF) → MPLS
ip -6 rule add from ::/0 to ::/0 \
    tos 0xb8 \
    lookup 200 \
    prio 100

# Rule 2: Traffic from VoIP VLAN → MPLS
ip -6 rule add from 2001:db8:voip::/64 \
    lookup 200 \
    prio 110

# Rule 3: Video traffic (DSCP AF41) → Broadband
ip -6 rule add from ::/0 to ::/0 \
    tos 0x88 \
    lookup 201 \
    prio 120

# Rule 4: Bulk/backup traffic → LTE
ip -6 rule add from 2001:db8:bulk::/64 \
    lookup 202 \
    prio 130

# Default: use main table
ip -6 rule add prio 32766 lookup main

# Show all rules
ip -6 rule show
ip -6 route show table 200
```

## nftables IPv6 Traffic Marking for PBR

```bash
# /etc/nftables-pbr.conf - Mark IPv6 packets for policy routing

table ip6 sd_wan_pbr {
    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # Mark VoIP RTP for MPLS path (mark 200)
        meta l4proto udp udp dport 10000-20000 meta mark set 0xc8 counter

        # Mark SIP signaling for MPLS path (mark 200)
        meta l4proto udp udp dport 5060 meta mark set 0xc8 counter
        meta l4proto tcp tcp dport 5060 meta mark set 0xc8 counter

        # Mark video streaming for broadband (mark 201)
        meta l4proto tcp tcp dport { 80, 443, 8080 } \
            ip6 dscp af41 meta mark set 0xc9 counter

        # Mark bulk transfers for LTE backup (mark 202)
        meta l4proto tcp tcp dport { 20, 21 } meta mark set 0xca counter
    }
}
```

```bash
# Load nftables and configure fwmark routing
sudo nft -f /etc/nftables-pbr.conf

# Add ip6tables rules to mark packets
ip6tables -t mangle -A PREROUTING -p udp --dport 10000:20000 -j MARK --set-mark 200
ip6tables -t mangle -A PREROUTING -p udp --dport 5060 -j MARK --set-mark 200

# Add routing rules based on fwmark
ip -6 rule add fwmark 200 lookup 200 prio 100
ip -6 rule add fwmark 201 lookup 201 prio 110
ip -6 rule add fwmark 202 lookup 202 prio 120
```

## Cisco IOS XE IPv6 Policy Routing

```text
! Create IPv6 access-lists for classification
ipv6 access-list VOIP-IPV6
 permit udp 2001:db8:lan::/64 any range 10000 20000 dscp ef
 permit udp 2001:db8:lan::/64 any eq 5060

ipv6 access-list VIDEO-IPV6
 permit tcp 2001:db8:lan::/64 any dscp af41

ipv6 access-list BULK-IPV6
 permit tcp 2001:db8:lan::/64 any eq 21

! Route maps for policy routing
route-map PBR-VOIP-IPV6 permit 10
 match ipv6 address VOIP-IPV6
 set ipv6 next-hop 2001:db8:mpls::gateway

route-map PBR-VIDEO-IPV6 permit 20
 match ipv6 address VIDEO-IPV6
 set ipv6 next-hop 2001:db8:broadband::gateway

route-map PBR-BULK-IPV6 permit 30
 match ipv6 address BULK-IPV6
 set ipv6 next-hop 2001:db8:lte::gateway

! Apply to LAN interface
interface GigabitEthernet0/1
 ipv6 address 2001:db8:lan::1/64
 ipv6 policy route-map PBR-VOIP-IPV6
 ipv6 policy route-map PBR-VIDEO-IPV6
 ipv6 policy route-map PBR-BULK-IPV6

! Verify
show ipv6 policy
show route-map PBR-VOIP-IPV6
```

## Juniper IPv6 Policy Routing

```text
# Juniper firewall filter for IPv6 PBR
set firewall family inet6 filter PBR-SD-WAN-IPV6 term VOIP-TRAFFIC from \
    destination-port [5060 10000-20000] dscp ef

set firewall family inet6 filter PBR-SD-WAN-IPV6 term VOIP-TRAFFIC then \
    routing-instance MPLS-VRF

set firewall family inet6 filter PBR-SD-WAN-IPV6 term VIDEO-TRAFFIC from \
    dscp af41

set firewall family inet6 filter PBR-SD-WAN-IPV6 term VIDEO-TRAFFIC then \
    routing-instance BROADBAND-VRF

set firewall family inet6 filter PBR-SD-WAN-IPV6 term DEFAULT then accept

# Apply to LAN interface
set interfaces ge-0/0/1 unit 0 family inet6 filter input PBR-SD-WAN-IPV6
```

## Verify IPv6 Policy Routing

```bash
# Verify rules are being matched
ip -6 rule show
# Expected:
# 100: from ::/0 to ::/0 tos 0xb8 lookup 200
# 110: from 2001:db8:voip::/64 lookup 200

# Test routing decision for specific IPv6 source
ip -6 route get 2001:db8:remote::1 from 2001:db8:voip::50
# Should show: via MPLS gateway, dev eth0

# Monitor policy route hits
ip6tables -t mangle -L PREROUTING -n -v | grep -v "^0\s"

# Trace packet path
tcptraceroute6 2001:db8:remote::1 80 -s 2001:db8:voip::50
```

IPv6 policy-based routing in SD-WAN combines Linux routing tables with ip6tables fwmark rules or router PBR features to steer traffic by source prefix, DSCP marking, or destination service, enabling fine-grained path selection that aligns IPv6 traffic with appropriate WAN links based on cost, performance, and application requirements.
