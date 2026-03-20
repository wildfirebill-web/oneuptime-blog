# How to Rate Limit IPv6 Connections with ip6tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ip6tables, Rate Limiting, DoS Protection, Linux

Description: Learn how to implement IPv6 connection rate limiting with ip6tables using the limit, hashlimit, and recent modules to protect against DoS attacks and brute force.

## Overview

Rate limiting with ip6tables restricts how many packets or connections can arrive per unit time. For IPv6, this is important for preventing: ICMPv6 floods, SSH brute-force attacks, SYN floods, and NDP exhaustion. ip6tables provides three main rate-limiting mechanisms: the `limit` module (global), `hashlimit` module (per-source), and `recent` module (connection tracking).

## Module 1: limit (Global Rate Limiting)

The `limit` module applies a rate limit globally across all sources:

```bash
# Limit incoming ping requests to 10 per second (global)
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request \
  -m limit --limit 10/second --limit-burst 30 \
  -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -j DROP

# Limit new SSH connections to 4 per minute
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m limit --limit 4/minute --limit-burst 8 \
  -j ACCEPT
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j DROP

# Limit SYN flood rate
ip6tables -A INPUT -p tcp --syn \
  -m limit --limit 100/second --limit-burst 500 \
  -j ACCEPT
ip6tables -A INPUT -p tcp --syn -j DROP
```

**limit parameters:**
- `--limit N/unit` — Allow N packets per unit (second, minute, hour)
- `--limit-burst B` — Initial burst allowance (tokens)

## Module 2: hashlimit (Per-Source Rate Limiting)

The `hashlimit` module applies rates per-source IP address:

```bash
# Limit SSH to 3 new connections per minute per source IPv6
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m hashlimit \
  --hashlimit-name ssh-rate-v6 \
  --hashlimit-upto 3/minute \
  --hashlimit-burst 5 \
  --hashlimit-mode srcip \
  --hashlimit-htable-expire 300000 \
  -j ACCEPT
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j DROP

# Limit HTTP new connections per source
ip6tables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW \
  -m hashlimit \
  --hashlimit-name http-rate-v6 \
  --hashlimit-upto 50/second \
  --hashlimit-burst 200 \
  --hashlimit-mode srcip \
  -j ACCEPT
ip6tables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j DROP
```

**hashlimit parameters:**
- `--hashlimit-name` — Unique name for hash table
- `--hashlimit-upto N/unit` — Allow up to N packets per unit
- `--hashlimit-burst B` — Burst allowance
- `--hashlimit-mode srcip` — Track per source IP (or srcport, dstip, dstport)

## Module 3: recent (Recent Connection Tracking)

The `recent` module tracks recent connections to implement more complex rules:

```bash
# SSH brute-force protection
# If 4+ connections from same source in 60 seconds → drop
ip6tables -A INPUT -p tcp --dport 22 \
  -m recent --name SSH6 --rcheck --seconds 60 --hitcount 4 \
  -j LOG --log-prefix "SSH-BRUTE-FORCE: "
ip6tables -A INPUT -p tcp --dport 22 \
  -m recent --name SSH6 --rcheck --seconds 60 --hitcount 4 \
  -j DROP
ip6tables -A INPUT -p tcp --dport 22 \
  -m recent --name SSH6 --set \
  -j ACCEPT
```

## NDP Exhaustion Protection

NDP exhaustion is an IPv6-specific attack where an attacker sends neighbor solicitations for thousands of addresses in a /64 subnet, forcing the router to send unreachable messages:

```bash
# Limit Neighbor Solicitations per source
ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 \
  -m hashlimit \
  --hashlimit-name ndp-limit \
  --hashlimit-upto 10/second \
  --hashlimit-burst 20 \
  --hashlimit-mode srcip \
  -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 -j DROP
```

## ICMPv6 Flood Protection

```bash
# Rate limit all ICMPv6 (except critical types already accepted)
ip6tables -A INPUT -p icmpv6 \
  -m limit --limit 20/second --limit-burst 50 \
  -j ACCEPT
ip6tables -A INPUT -p icmpv6 -j DROP

# More granular: per-source ICMPv6 limiting
ip6tables -A INPUT -p icmpv6 \
  -m hashlimit \
  --hashlimit-name icmpv6-limit \
  --hashlimit-upto 5/second \
  --hashlimit-burst 10 \
  --hashlimit-mode srcip \
  -j ACCEPT
ip6tables -A INPUT -p icmpv6 -j DROP
```

## Log Rate-Limited Drops

```bash
# Log rate-limited drops for alerting
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m hashlimit \
  --hashlimit-name ssh-log \
  --hashlimit-upto 3/minute --hashlimit-burst 5 \
  --hashlimit-mode srcip \
  -j LOG --log-prefix "SSH-RATE-LIMIT: "

ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j DROP
```

## View hashlimit Tables

```bash
# View current hashlimit entries (Linux 5.x+)
cat /proc/net/ip6_tables_matches | grep hashlimit
# or
nft list table ip6 filter 2>/dev/null | grep hashlimit

# hashlimit entries are in /proc/net/ipt_hashlimit/ (ipv4) or similar
ls /proc/net/ | grep hashlimit
```

## Summary

ip6tables rate limiting uses three modules: `limit` for global rates (all sources combined), `hashlimit` for per-source rates (most useful — limits each attacker individually), and `recent` for connection-count-based blocking (SSH brute force). For IPv6, always rate-limit ICMPv6 echo requests, new SSH connections, new HTTP connections, and Neighbor Solicitations (to prevent NDP exhaustion). The `hashlimit` module with `--hashlimit-mode srcip` is the most effective for per-source protection. Log rate-limited drops to detect ongoing attacks.
