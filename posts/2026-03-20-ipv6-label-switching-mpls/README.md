# How IPv6 Label Switching Works in MPLS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MPLS, Label Switching, LDP, RSVP, LSP, Forwarding, Data Plane

Description: Understand how MPLS label switching handles IPv6 packets including label allocation for IPv6 prefixes, label stacking in 6PE/6VPE, and MPLS forwarding plane behavior for IPv6 traffic.

---

MPLS label switching for IPv6 operates identically to IPv4 at the forwarding plane—routers switch packets based on 32-bit labels without examining the IP header. IPv6 enters MPLS through labeled BGP routes (6PE) or via RSVP-TE IPv6 tunnels, with the outer label controlling forwarding.

## MPLS Label Structure

```
MPLS Label Stack Entry (32 bits):
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Label (20 bits)                |EXP|S|  TTL  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Label: 0-1048575 (labels 0-15 reserved)
EXP: Experimental (now TC - Traffic Class for QoS)
S: Bottom of Stack (1 = last label in stack)
TTL: Time to Live

For 6PE label stack:
[LDP Transport Label (S=0)] [BGP IPv6 Label (S=1)] [IPv6 Packet]
```

## LDP Label Distribution for IPv6

```bash
# LDP distributes labels for IPv4 prefixes in the routing table
# For 6PE, LDP provides transport labels; BGP provides IPv6 labels

# Check LDP label bindings (IPv4 - transport labels)
show mpls ldp bindings

# On Cisco IOS:
show mpls ldp bindings | include 10.0.0.2
# In-label  Out-label  Prefix     Nexthop
# 16        16         10.0.0.2   10.1.1.2  ← transport label

# For BGP-distributed IPv6 labels:
show bgp ipv6 unicast labels
# Network         Next Hop     In label/Out label
# 2001:db8:b::/48 10.0.0.2     17/16

# Full label stack for packet to 2001:db8:b::10:
# [LDP label 16 for 10.0.0.2] [BGP label 17 for 2001:db8:b::/48]
```

## MPLS Forwarding for IPv6

```
IPv6 Packet Forwarding in 6PE MPLS:

Ingress PE (PE1):
  IPv6 packet arrives from CE: dst=2001:db8:b::10
  Lookup in inet6.0 (or CEF IPv6 table)
  Find: 2001:db8:b::/48 → BGP label 17, nexthop PE2 (10.0.0.2)
  Find: 10.0.0.2 → LDP label 16, interface ge0/0
  Push labels: [16][17] + IPv6 packet
  Forward out ge0/0

P Router (Core):
  See label 16 in forwarding table
  Swap to label 17 (toward PE2)
  Forward to next hop

Last P Router (PHP - Penultimate Hop Popping):
  See label 17 → PHP enabled → pop label
  Forward to PE2 with label 17... or
  If PHP: send raw [17][IPv6] to PE2

Egress PE (PE2):
  Receive [17] + IPv6
  Pop label 17
  Lookup IPv6 dst in local table
  Forward to CE

EXP/TC Bits in MPLS Header:
  Copy IPv6 DSCP/Traffic Class to MPLS EXP bits for QoS
  tc dscp ef (0x2e) → EXP = 5 or 7
```

## RSVP-TE for IPv6 MPLS Tunnels

```bash
# Configure RSVP-TE tunnel for IPv6 over MPLS

# Cisco IOS - RSVP-TE tunnel for IPv6 traffic
interface Tunnel0
 description IPv6-TE-Tunnel-to-PE2
 ip unnumbered Loopback0
 tunnel destination 10.0.0.2
 tunnel mode mpls traffic-eng
 tunnel mpls traffic-eng autoroute announce
 tunnel mpls traffic-eng bandwidth 100000
 tunnel mpls traffic-eng path-option 1 explicit name TO-PE2

! Route IPv6 traffic over TE tunnel
ip explicit-path name TO-PE2
 next-address 10.1.1.2
 next-address 10.2.2.2

! Apply IPv6 traffic to TE tunnel
ipv6 route 2001:db8:site-b::/48 Tunnel0
```

## Segment Routing (SR-MPLS) for IPv6

```bash
# Modern approach: SR-MPLS with IPv6 prefixes

# Cisco IOS XE - Segment Routing with IPv6
segment-routing mpls

# Enable SR on OSPF (ISIS also supports SR)
router ospf 1
 segment-routing mpls
 segment-routing prefix-sid-map advertise-local

! Configure SID (Segment ID = MPLS label) for loopback
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
 ip ospf prefix-sid index 1  ! Prefix SID: base-label + 1

! IPv6 routes get SR labels via BGP
router bgp 65000
 address-family ipv6
  neighbor 10.0.0.2 send-label

! Check SR forwarding
show segment-routing mpls connected-prefix-sid-map
show mpls forwarding-table
```

## Monitor MPLS Label Usage for IPv6

```bash
# Check label space utilization
show mpls label range
show mpls label table

# View LFIB (Label Forwarding Information Base) for IPv6
show mpls forwarding-table detail | grep -A3 "IPv6\|inet6"

# Juniper: Check label space for inet6-vpn
show route table bgp.l3vpn-inet6.0 | head -20

# Check label allocation by protocol
show mpls label table | grep -E "bgp|ospf|ldp"

# Monitor MPLS IPv6 packet counters
show interface GigabitEthernet0/0 counters | include MPLS
```

MPLS label switching for IPv6 is protocol-agnostic at the data plane—P routers forward based solely on the 32-bit label regardless of whether the payload is IPv4, IPv6, or anything else—with 6PE and 6VPE requiring only the ingress PE to perform the initial IPv6-to-label mapping while all intermediate P routers perform fast label switching without IPv6 awareness.
