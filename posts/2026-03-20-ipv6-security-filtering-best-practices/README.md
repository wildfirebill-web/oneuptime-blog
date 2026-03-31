# How to Understand IPv6 Security Filtering Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Filtering, Firewall, Best Practice

Description: Learn the comprehensive best practices for IPv6 traffic filtering at both perimeter and host levels, including ICMPv6 policy, bogon filtering, and extension header handling.

## Overview

IPv6 filtering best practices differ from IPv4 in key ways: ICMPv6 is integral to core protocol functions (NDP, PMTUD), some header fields have no IPv4 equivalent (Flow Label, extension headers), and first-hop security is more complex. This guide provides a complete framework for IPv6 filtering based on RFC 4890 and operational experience.

## RFC 4890: ICMPv6 Filtering Recommendations

RFC 4890 is the authoritative guide on which ICMPv6 messages to allow and which to block:

### Must Not Drop (Critical Functionality)

| Type | Name | Why Critical |
|------|------|-------------|
| 1 | Destination Unreachable | Path failure notification |
| 2 | Packet Too Big | Required for PMTUD - must never be blocked |
| 3 | Time Exceeded | Traceroute, loop detection |
| 4 | Parameter Problem | Malformed packet notification |
| 128 | Echo Request | Reachability testing (allow selectively) |
| 129 | Echo Reply | Response to echo request |
| 133 | Router Solicitation | SLAAC - link-local only |
| 134 | Router Advertisement | SLAAC - link-local only, filter at perimeter |
| 135 | Neighbor Solicitation | NDP - link-local only |
| 136 | Neighbor Advertisement | NDP - link-local only |

### May Block at Perimeter

| Type | Name | Policy |
|------|------|--------|
| 130-132 | Multicast Listener Discovery | Block unless multicast in use |
| 143 | MLDv2 | Block from internet |
| 144-147 | Mobile IPv6 | Block unless Mobile IPv6 deployed |
| 148-149 | SEND | Block unless SEND deployed |

## Complete ip6tables Best Practice Policy

```bash
#!/bin/bash
# IPv6 best practice filtering policy

# Flush existing rules

ip6tables -F
ip6tables -X

# Default policies
ip6tables -P INPUT   DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT  ACCEPT

# Allow loopback
ip6tables -A INPUT -i lo -j ACCEPT

# Allow established and related connections
ip6tables -A INPUT  -m state --state ESTABLISHED,RELATED -j ACCEPT
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# === ICMPv6 - Critical (RFC 4890) ===
# Packet Too Big - MUST allow (PMTUD)
ip6tables -A INPUT   -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Destination Unreachable
ip6tables -A INPUT   -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT

# Time Exceeded
ip6tables -A INPUT   -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT

# Parameter Problem
ip6tables -A INPUT   -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT

# NDP - link-local only
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type router-solicitation -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-solicitation -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-advertisement -j ACCEPT

# Echo (ping) - allow, optionally rate-limit
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -m limit --limit 10/s -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-reply -j ACCEPT

# Block all other ICMPv6
ip6tables -A INPUT -p icmpv6 -j DROP

# === Bogon Source Filtering ===
ip6tables -A INPUT -s ::/128 -j DROP
ip6tables -A INPUT -s ::1/128 -j DROP
ip6tables -A INPUT -s ::ffff:0:0/96 -j DROP
ip6tables -A INPUT -s 2001:db8::/32 -j DROP
ip6tables -A INPUT -s fc00::/7 -j DROP
ip6tables -A INPUT -s fe80::/10 -j DROP   # (except NDP above)

# === Extension Headers ===
# Block Routing Header Type 0 (deprecated)
ip6tables -A INPUT   -m rt --rt-type 0 -j DROP
ip6tables -A FORWARD -m rt --rt-type 0 -j DROP

# === Services ===
ip6tables -A INPUT -p tcp --dport 22  -j ACCEPT
ip6tables -A INPUT -p tcp --dport 80  -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# === Logging before final drop ===
ip6tables -A INPUT   -j LOG --log-prefix "IPv6-IN-DROP: "
ip6tables -A FORWARD -j LOG --log-prefix "IPv6-FWD-DROP: "
```

## nftables Best Practice Policy

```bash
#!/usr/sbin/nft -f
# nftables IPv6 best practice

table ip6 filter {
    chain input {
        type filter hook input priority 0; policy drop;

        iif lo accept
        ct state established,related accept

        # ICMPv6 critical
        ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem } accept
        ip6 saddr fe80::/10 ip6 nexthdr icmpv6 icmpv6 type { nd-neighbor-solicit, nd-neighbor-advert, nd-router-solicit, nd-router-advert } accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate 10/second accept
        ip6 nexthdr icmpv6 icmpv6 type echo-reply accept
        ip6 nexthdr icmpv6 drop

        # Bogon sources
        ip6 saddr { ::/128, ::1/128, 2001:db8::/32, fc00::/7 } drop

        # Services
        tcp dport { 22, 80, 443 } accept

        log prefix "IPv6-DROP: "
    }
}
```

## Perimeter vs Host Filtering

| Control | Perimeter Firewall | Host Firewall |
|---------|-------------------|---------------|
| Bogon filtering | Yes - at ingress | No (already filtered) |
| ICMPv6 policy | Allow PTB + unreachable + NDP | Same + rate-limit echo |
| Extension headers | Block RH0, audit others | Block RH0 |
| NDP (RS/RA/NS/NA) | Block from internet | Allow from link-local only |
| Stateful inspection | Yes | Yes |

## Summary

IPv6 filtering best practices require: (1) never blocking Packet Too Big (type 2), which breaks PMTUD; (2) allowing NDP from link-local sources only - block from internet; (3) filtering bogon prefixes at ingress; (4) blocking Routing Header Type 0; (5) logging drops for SIEM analysis. Use the ip6tables or nftables templates as a starting point and adapt to your service requirements. Review against RFC 4890 for ICMPv6 policy guidance.
