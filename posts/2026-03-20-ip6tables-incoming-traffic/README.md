# How to Write ip6tables Rules for Incoming IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ip6tables, Firewall, Linux, Incoming Traffic

Description: Learn how to write comprehensive ip6tables rules for controlling incoming IPv6 traffic, including per-service rules, source restrictions, and rate limiting.

## Overview

Incoming IPv6 traffic is controlled via the INPUT chain in ip6tables. A well-designed INPUT policy uses a default DROP policy and explicitly allows only the traffic your system needs. This guide covers patterns for controlling access to common services, restricting by source prefix, and protecting against common attacks.

## Default Policy: Deny All Inbound

```bash
# Start with DROP all — then explicitly allow what's needed
ip6tables -P INPUT DROP
```

This means any traffic not explicitly matched by a rule is dropped.

## Essential Rules for Any Server

```bash
# 1. Loopback — always allow
ip6tables -A INPUT -i lo -j ACCEPT

# 2. Established connections — allow replies
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 3. Invalid packets — drop immediately
ip6tables -A INPUT -m conntrack --ctstate INVALID -j DROP

# 4. Essential ICMPv6 — MUST allow (RFC 4890)
# Packet Too Big — required for PMTUD (never block this)
ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT

# Destination Unreachable (type 1)
ip6tables -A INPUT -p icmpv6 --icmpv6-type 1 -j ACCEPT

# Time Exceeded (type 3)
ip6tables -A INPUT -p icmpv6 --icmpv6-type 3 -j ACCEPT

# Parameter Problem (type 4)
ip6tables -A INPUT -p icmpv6 --icmpv6-type 4 -j ACCEPT

# NDP from link-local only (types 133-137)
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 133 -j ACCEPT  # Router Solicitation
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 134 -j ACCEPT  # Router Advertisement
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 135 -j ACCEPT  # Neighbor Solicitation
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 136 -j ACCEPT  # Neighbor Advertisement
```

## Service-Specific Rules

### SSH

```bash
# Allow SSH from everywhere (basic)
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow SSH only from management network (better)
ip6tables -A INPUT -p tcp --dport 22 -s fd00:mgmt::/48 -j ACCEPT

# Allow SSH with connection limiting (prevent brute force)
ip6tables -A INPUT -p tcp --dport 22 -m recent --name SSH --rcheck --seconds 60 --hitcount 4 -j DROP
ip6tables -A INPUT -p tcp --dport 22 -m recent --name SSH --set -j ACCEPT
```

### HTTP and HTTPS

```bash
# Allow web traffic from anyone
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# Rate limit new HTTP connections
ip6tables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW \
          -m limit --limit 100/second --limit-burst 200 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j DROP
```

### DNS Server

```bash
# Allow DNS queries (UDP and TCP)
ip6tables -A INPUT -p udp --dport 53 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 53 -j ACCEPT
```

### Mail Server

```bash
# SMTP inbound (port 25)
ip6tables -A INPUT -p tcp --dport 25 -j ACCEPT
# Submission (port 587) — from authorized users only
ip6tables -A INPUT -p tcp --dport 587 -j ACCEPT
# IMAP/POP
ip6tables -A INPUT -p tcp --dport 993 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 995 -j ACCEPT
```

### Database (Restrict by Source)

```bash
# PostgreSQL — only from application servers
ip6tables -A INPUT -p tcp --dport 5432 -s 2001:db8:app::/64 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 5432 -j DROP   # Deny all others explicitly
```

## Rate Limiting Rules

```bash
# Limit incoming pings to prevent ICMP flood
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request \
          -m limit --limit 10/second --limit-burst 30 -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -j DROP

# Limit new TCP connection rate (SYN flood protection)
ip6tables -A INPUT -p tcp --syn -m limit --limit 50/second --limit-burst 100 -j ACCEPT
ip6tables -A INPUT -p tcp --syn -j DROP
```

## Blocking Specific Sources

```bash
# Block a specific attacker IPv6 address
ip6tables -A INPUT -s 2001:db8:bad::/48 -j DROP

# Block bogon sources (should be at top of INPUT)
ip6tables -I INPUT 1 -s ::/128 -j DROP
ip6tables -I INPUT 2 -s 2001:db8::/32 -j DROP
ip6tables -I INPUT 3 -s fc00::/7 -j DROP
```

## Logging Incoming Drops

```bash
# Log with rate limiting to prevent log flooding
ip6tables -A INPUT -m limit --limit 5/minute \
          -j LOG --log-prefix "IPv6-INPUT-DROP: " --log-level 4
ip6tables -A INPUT -j DROP
```

## Complete Incoming Policy Template

```bash
#!/bin/bash
# Flush INPUT chain
ip6tables -F INPUT
ip6tables -P INPUT DROP

# 1. Loopback
ip6tables -A INPUT -i lo -j ACCEPT

# 2. Established
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A INPUT -m conntrack --ctstate INVALID -j DROP

# 3. Bogon sources
for BOGON in '::/128' '::1/128' '2001:db8::/32' 'fc00::/7'; do
  ip6tables -A INPUT -s $BOGON -j DROP
done

# 4. Critical ICMPv6
ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT    # Packet Too Big
ip6tables -A INPUT -p icmpv6 --icmpv6-type 1 -j ACCEPT    # Unreachable
ip6tables -A INPUT -p icmpv6 --icmpv6-type 3 -j ACCEPT    # Time Exceeded
ip6tables -A INPUT -p icmpv6 --icmpv6-type 4 -j ACCEPT    # Param Problem
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 133 -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 134 -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 135 -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 136 -j ACCEPT

# 5. Services
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# 6. Log and drop remainder
ip6tables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPv6-IN-DROP: "
```

## Summary

ip6tables INPUT rules for IPv6 follow the pattern: allow loopback, allow established connections, drop invalid, allow required ICMPv6 (types 1, 2, 3, 4 and NDP from fe80::/10), then allow specific services, and drop everything else. NDP (types 133-137) must only be allowed from link-local sources to prevent rogue RA injection. Rate-limit ICMPv6 echo requests and new TCP connections to prevent floods. Always log drops with a rate limit to keep log volume manageable.
