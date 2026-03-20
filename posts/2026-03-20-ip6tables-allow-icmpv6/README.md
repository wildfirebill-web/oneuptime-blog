# How to Configure ip6tables to Allow Essential ICMPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ip6tables, ICMPv6, Firewall, RFC 4890

Description: Learn which ICMPv6 types are essential and must never be blocked, which can be filtered at the perimeter, and how to write correct ip6tables rules following RFC 4890.

## Overview

ICMPv6 is far more important in IPv6 than ICMP was in IPv4. Many critical IPv6 functions - Neighbor Discovery Protocol (NDP), Path MTU Discovery (PMTUD), and stateless address autoconfiguration (SLAAC) - depend on specific ICMPv6 message types. Incorrectly blocking ICMPv6 can break connectivity in subtle ways that are hard to diagnose.

## ICMPv6 Types Reference

| Type | Name | Function | Policy |
|------|------|----------|--------|
| 1 | Destination Unreachable | Path failure notification | MUST allow |
| 2 | Packet Too Big | Path MTU Discovery | MUST NEVER block |
| 3 | Time Exceeded | TTL/loop detection, traceroute | MUST allow |
| 4 | Parameter Problem | Malformed packet notification | MUST allow |
| 128 | Echo Request | Ping | Allow (admin discretion) |
| 129 | Echo Reply | Ping response | Allow with 128 |
| 133 | Router Solicitation | SLAAC | Allow from link-local only |
| 134 | Router Advertisement | SLAAC | Allow from link-local only |
| 135 | Neighbor Solicitation | NDP (like ARP) | Allow from link-local only |
| 136 | Neighbor Advertisement | NDP (like ARP reply) | Allow from link-local only |
| 137 | Redirect | Next-hop optimization | Allow from link-local only |
| 143 | MLDv2 Report | Multicast membership | Allow on LAN only |
| 130-132 | MLD | Multicast Listener Discovery | Allow on LAN, block at perimeter |
| 144-147 | Mobile IPv6 | Home agent, BU | Block unless Mobile IPv6 used |

## Why Packet Too Big (Type 2) Must NEVER Be Blocked

IPv6 requires Path MTU Discovery (PMTUD, RFC 8201). Routers in IPv6 never fragment packets - they return a "Packet Too Big" ICMPv6 message to the sender, which then reduces its packet size.

If type 2 is blocked:
- Large TCP transfers silently fail after the 3-way handshake
- "Works from ping but not for large files" is a classic PMTUD symptom
- VoIP, video streaming, and any large-packet application breaks

```bash
# This rule MUST be present - NEVER remove it

ip6tables -A INPUT   -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A OUTPUT  -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
```

## Why NDP Must Only Come From Link-Local

NDP messages (types 133-137) are used for:
- Address resolution (neighbor solicitation/advertisement)
- Router discovery (router solicitation/advertisement)

These should only come from directly connected hosts (link-local scope, fe80::/10):

```bash
# CORRECT: Allow NDP only from link-local
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-solicitation -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-advertisement -j ACCEPT

# WRONG: Allowing NDP from anywhere - enables rogue RA attacks
# ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
```

## Complete ICMPv6 Policy (RFC 4890 Compliant)

```bash
#!/bin/bash
# Complete ICMPv6 ip6tables policy (RFC 4890 compliant)

# ===== Critical - MUST allow (all directions) =====

# Destination Unreachable (all codes)
ip6tables -A INPUT   -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
ip6tables -A OUTPUT  -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT

# Packet Too Big - NEVER block
ip6tables -A INPUT   -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A OUTPUT  -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Time Exceeded
ip6tables -A INPUT   -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
ip6tables -A OUTPUT  -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT

# Parameter Problem
ip6tables -A INPUT   -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT
ip6tables -A OUTPUT  -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT

# ===== NDP - link-local only =====
for NDPTYPE in router-solicitation router-advertisement neighbour-solicitation neighbour-advertisement; do
    ip6tables -A INPUT  -s fe80::/10 -p icmpv6 --icmpv6-type $NDPTYPE -j ACCEPT
    ip6tables -A OUTPUT -p icmpv6 --icmpv6-type $NDPTYPE -j ACCEPT
done

# ===== Echo - allow inbound with rate limit =====
ip6tables -A INPUT  -p icmpv6 --icmpv6-type echo-request \
          -m limit --limit 10/second --limit-burst 30 -j ACCEPT
ip6tables -A INPUT  -p icmpv6 --icmpv6-type echo-reply -j ACCEPT
ip6tables -A OUTPUT -p icmpv6 --icmpv6-type echo-request -j ACCEPT
ip6tables -A OUTPUT -p icmpv6 --icmpv6-type echo-reply -j ACCEPT

# ===== Block everything else =====
ip6tables -A INPUT   -p icmpv6 -j DROP
ip6tables -A FORWARD -p icmpv6 -j DROP
```

## Verifying ICMPv6 Policy

```bash
# Test Packet Too Big handling
# From remote host, send large packets:
ping6 -c 3 -s 1500 your-host.example.com
# If this fails while small pings work → PTB is blocked

# Verify NDP works (neighbor resolution)
ip -6 neigh show   # Should show entries for local neighbors

# Test traceroute (requires Time Exceeded to be allowed)
traceroute6 -n 2001:db8::1
# Should show hops, not "* * *" (which would indicate time-exceeded is blocked)
```

## Summary

ICMPv6 requires careful filtering: never block Packet Too Big (type 2 - breaks PMTUD), Destination Unreachable (type 1), Time Exceeded (type 3), or Parameter Problem (type 4). Allow NDP types (133-137) only from link-local sources (fe80::/10) to prevent rogue Router Advertisement attacks. Rate-limit Echo Request (type 128) to prevent ICMP floods. Drop all other ICMPv6 types at the perimeter including MLD (130-132) from the internet. Follow RFC 4890 for complete guidance on the correct ICMPv6 filtering policy.
