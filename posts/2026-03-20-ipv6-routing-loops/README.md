# How to Troubleshoot IPv6 Routing Loops

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Routing, Troubleshooting, traceroute6, Network Diagnostics, BGP

Description: Diagnose IPv6 routing loops using traceroute6, analyze TTL-exceeded messages, and identify misconfigured static routes or dynamic routing protocol issues causing loops.

## Introduction

IPv6 routing loops occur when packets cycle between two or more routers indefinitely. The IPv6 hop limit (equivalent to IPv4 TTL) decrements at each hop and the packet is dropped when it reaches 0, generating an ICMPv6 "Time Exceeded" message. Routing loops waste bandwidth, cause high latency, and prevent traffic from reaching its destination.

## Step 1: Detect a Routing Loop with traceroute6

```bash
# Run traceroute6 to detect loops
traceroute6 2001:db8::1

# A routing loop shows the same hops repeating:
# 1  2001:db8::fe01   1.234ms
# 2  2001:db8::fe02   2.345ms
# 3  2001:db8::fe01   3.456ms  ← same as hop 1 (LOOP!)
# 4  2001:db8::fe02   4.567ms
# ... continues until hop limit exhausted

# Use mtr for continuous loop detection
mtr --ipv6 2001:db8::1

# Set high hop limit to see full loop cycle
traceroute6 -m 50 2001:db8::1
```

## Step 2: Check Local Routing Table

```bash
# Show the full IPv6 routing table
ip -6 route show

# Check if two routes point to each other (recursive loop)
ip -6 route show | grep default

# Example of a loop-causing configuration:
# default via 2001:db8::1 dev eth0  (host points to router A)
# At router A: default via 2001:db8::100  (router A points back to host!)

# Check route for specific destination
ip -6 route get 2001:db8::external

# Show all routes sorted by destination
ip -6 route show | sort
```

## Step 3: Check for Static Route Loops

```bash
# List all static routes
ip -6 route show proto static

# Check for mutually pointing static routes:
# Router A: ip -6 route add 2001:db8:b::/64 via fe80::router-b dev eth1
# Router B: ip -6 route add 2001:db8:a::/64 via fe80::router-a dev eth0
# If these are both "catch-all" defaults that are too broad, loop occurs

# Verify next-hop is reachable and not a loopback
ip -6 route get fe80::router-b
ndisc6 fe80::router-b eth1
```

## Step 4: Check Dynamic Routing (OSPFv3/BGP)

```bash
# Check OSPFv3 routes with FRRouting
vtysh -c "show ipv6 route"
vtysh -c "show ipv6 ospf6 route"

# Check BGP routes
vtysh -c "show bgp ipv6 unicast"
vtysh -c "show bgp ipv6 unicast summary"

# Identify routes with same next-hop pointing back
vtysh -c "show ipv6 route" | grep -A2 "::/0"

# Check for BGP route reflector loops (ensure no-export communities)
vtysh -c "show bgp ipv6 unicast community no-export"
```

## Step 5: Fix a Routing Loop

```bash
# Scenario: default route loop between two routers

# Router A incorrectly has:
# default via fe80::router-b dev eth0

# Router B incorrectly has:
# default via fe80::router-a dev eth0

# Fix: Only one should have the default route pointing "upward"
# The other should have a specific route, not a default

# On Router A (correct gateway for external traffic):
ip -6 route del default via fe80::router-b dev eth0
ip -6 route add default via 2001:db8::isp-router dev eth1

# On Router B (should use Router A for external):
ip -6 route del default via fe80::router-a dev eth0
ip -6 route add default via 2001:db8::router-a dev eth0

# Verify no loop
traceroute6 2001:4860:4860::8888
```

## Step 6: Monitor for Loops with Packet Analysis

```bash
# Capture ICMPv6 Time Exceeded (type 3) — indicates hop limit expired in loop
sudo tcpdump -i eth0 -v "ip6 proto 58 and ip6[40] == 3"

# High rate of Time Exceeded messages indicates active routing loop

# Check kernel drop counters
cat /proc/net/snmp6 | grep "^Ip6.*Discard\|HopLimit"

# Show ICMP6 statistics
ip -6 -s route show | grep "Rt6Stats"
```

## Prevention: Use Route Metrics and Preferences

```bash
# Use metrics to prefer direct routes over default
ip -6 route add 2001:db8:a::/64 dev eth0 metric 100
ip -6 route add default via fe80::gw dev eth0 metric 200

# For iBGP: Use MED and local preference to prevent loops
# For OSPFv3: Ensure consistent costs and no bi-directional redistribution

# Enable ECMP (Equal-Cost Multi-Path) safely
# Ensure all ECMP paths lead to the same external destination
```

## Conclusion

IPv6 routing loops are detected by `traceroute6` showing repeated hops and diagnosed by comparing routing tables on all devices in the path. Static route loops occur when two routers point to each other as default gateways. Dynamic routing loops typically involve redistribution between protocols (BGP to OSPFv3 and back). Fix by ensuring only one path through the network for each destination prefix — no mutual default routes, and careful redistribution policies. Monitor with ICMPv6 Time Exceeded capture to detect active loops in production.
