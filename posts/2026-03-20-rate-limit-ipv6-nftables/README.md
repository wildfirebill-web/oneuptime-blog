# How to Rate Limit IPv6 Connections with nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, nftables, Rate Limiting, DoS Protection, Linux

Description: Learn how to implement IPv6 connection rate limiting with nftables using limit rate, meter, and flow table statements for protection against floods and brute force attacks.

## Overview

nftables provides more powerful and flexible rate limiting than ip6tables. The `limit rate` statement applies global rate limits, while `meter` and `flow table` statements enable per-source tracking. For IPv6, rate limiting is essential for protecting against ICMPv6 floods, NDP exhaustion, SSH brute force, and SYN flood attacks.

## Method 1: limit rate (Global)

```bash
# Limit incoming IPv6 ping rate globally
nft add rule ip6 filter input \
  ip6 nexthdr icmpv6 icmpv6 type echo-request \
  limit rate 10/second burst 30 packets accept

nft add rule ip6 filter input \
  ip6 nexthdr icmpv6 icmpv6 type echo-request drop

# Limit new SSH connections globally
nft add rule ip6 filter input \
  tcp dport 22 ct state new \
  limit rate 10/minute burst 20 packets accept

nft add rule ip6 filter input \
  tcp dport 22 ct state new drop
```

## Method 2: meter (Per-Source, Dynamic)

```bash
# Limit SSH per source IPv6 address (5 connections per minute)
nft add rule ip6 filter input \
  tcp dport 22 ct state new \
  meter ssh-limit { ip6 saddr timeout 60s limit rate 5/minute } accept

# If rate exceeded, no rule matches → falls through to default DROP
```

### Full Ruleset with meters

```bash
# /etc/nftables.d/ipv6-rate-limits.nft

table ip6 rate-limits {
    chain input {
        type filter hook input priority filter - 1;

        # Per-source SSH rate limit
        tcp dport 22 ct state new \
            meter ssh6-limit { ip6 saddr timeout 60s limit rate 4/minute } accept
        tcp dport 22 ct state new drop

        # Per-source HTTP rate limit
        tcp dport 80 ct state new \
            meter http6-limit { ip6 saddr timeout 10s limit rate 50/second burst 100 packets } accept

        # ICMPv6 echo rate limit per source
        ip6 nexthdr icmpv6 icmpv6 type echo-request \
            meter ping6-limit { ip6 saddr timeout 5s limit rate 5/second } accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request drop

        # NDP rate limit (NS per source — prevent NDP exhaustion)
        ip6 nexthdr icmpv6 icmpv6 type nd-neighbor-solicit \
            meter ndp-limit { ip6 saddr timeout 10s limit rate 10/second } accept
        ip6 nexthdr icmpv6 icmpv6 type nd-neighbor-solicit drop
    }
}
```

## Method 3: flow table (Conntrack-based Accounting)

Flow tables accelerate connection handling and can be used for rate tracking:

```bash
# Define a flow table for IPv6
table ip6 filter {
    flowtable ipv6-flow {
        hook ingress priority 0;
        devices = { eth0 };
    }

    chain forward {
        type filter hook forward priority 0;
        # Offload established flows to flow table
        ip6 nexthdr { tcp, udp } flow add @ipv6-flow
    }
}
```

## SYN Flood Protection

```bash
# Limit SYN packets (new TCP connections) to prevent SYN flood
nft add rule ip6 filter input \
  ip6 nexthdr tcp tcp flags syn \
  limit rate 100/second burst 500 packets accept

nft add rule ip6 filter input \
  ip6 nexthdr tcp tcp flags syn drop
```

## Log Rate-Limited Drops

```bash
# Log with rate limit to avoid log flooding
nft add rule ip6 filter input \
  tcp dport 22 ct state new \
  limit rate 1/second \
  log prefix "SSH-RATE-EXCEEDED: " level warn

nft add rule ip6 filter input \
  tcp dport 22 ct state new drop
```

## Complete Rate-Limiting Ruleset

```bash
#!/usr/sbin/nft -f
# IPv6 rate limiting configuration

flush table ip6 rate-control
table ip6 rate-control {

    chain input {
        type filter hook input priority filter - 1;

        # --- ICMPv6 Rate Limits ---
        # Echo request: 10/s burst 30, per source
        ip6 nexthdr icmpv6 icmpv6 type echo-request \
            meter ping-rate { ip6 saddr timeout 10s limit rate 10/second burst 30 packets } accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request \
            limit rate 5/minute log prefix "PING-FLOOD: " level warn
        ip6 nexthdr icmpv6 icmpv6 type echo-request drop

        # NDP NS: 10/s burst 20, per source
        ip6 saddr fe80::/10 ip6 nexthdr icmpv6 icmpv6 type nd-neighbor-solicit \
            meter ndp-rate { ip6 saddr timeout 10s limit rate 10/second } accept
        ip6 nexthdr icmpv6 icmpv6 type nd-neighbor-solicit \
            log prefix "NDP-EXHAUST: " level crit drop

        # --- TCP Rate Limits ---
        # SSH: 4/min per source
        tcp dport 22 ct state new \
            meter ssh-rate { ip6 saddr timeout 60s limit rate 4/minute burst 6 packets } accept
        tcp dport 22 ct state new \
            limit rate 10/second log prefix "SSH-BRUTE: " level warn drop

        # HTTP: 100/s per source
        tcp dport 80 ct state new \
            meter http-rate { ip6 saddr timeout 10s limit rate 100/second burst 500 packets } accept

        # SYN flood
        tcp flags syn ct state new \
            limit rate 500/second burst 1000 packets accept
        tcp flags syn ct state new \
            log prefix "SYN-FLOOD: " level crit drop
    }
}
```

## View Active Meters

```bash
# List defined meters and their state
nft list meters ip6 rate-control

# Sample output:
# table ip6 rate-control {
#     meter ssh-rate {
#         type ipv6_addr
#         size 65535
#         elements = {
#             2001:db8:1::100 : 2/minute : last used 3s ago
#             2001:db8:1::200 : 4/minute : last used 1s ago
#         }
#     }
# }

# Flush a specific meter
nft flush meter ip6 rate-control ssh-rate
```

## Summary

nftables rate limiting uses `limit rate N/unit burst B` for global limits and `meter name { ip6 saddr limit rate N/unit }` for per-source limits. Meters are more powerful than ip6tables `hashlimit` — they're defined inline in rules without separate initialization. For IPv6, apply per-source rate limits to: SSH (4-10/minute), HTTP (50-100/second), ICMPv6 echo (10/second), and Neighbor Solicitations (10/second) to prevent NDP exhaustion. Use `nft list meters` to monitor active connections against rate limits.
