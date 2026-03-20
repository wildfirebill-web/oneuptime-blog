# How to Filter IPv6 Extension Headers in Firewalls

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Firewalls, Extension Headers, Security, ip6tables

Description: Learn how to correctly filter IPv6 extension headers in firewalls, balancing security requirements with operational necessity by following RFC 7045 and RFC 4890 guidelines.

## Introduction

Firewall configuration for IPv6 extension headers requires careful balance: blocking too aggressively breaks legitimate traffic (fragmentation, IPsec, MLD), while allowing everything creates potential security exposure from deprecated headers like Routing Header Type 0. This guide provides practical firewall rules that follow RFC 7045 and RFC 4890 guidelines.

## Essential Policies for IPv6 Extension Headers

```
MUST allow (connectivity depends on them):
  NH=44 (Fragment): Allow for legitimate fragmented traffic
  NH=50 (ESP):      Allow for IPsec VPNs
  NH=51 (AH):       Allow for IPsec authentication
  NH=0  (HbH):      Allow for MLD and critical protocols
  NH=58 (ICMPv6):   Essential for NDP, Path MTU, etc.

MUST block:
  NH=43, RT Type 0 (Routing Header Type 0): Security vulnerability, deprecated

SHOULD examine carefully:
  NH=43 (Routing Header, non-Type 0): Allow Type 2, Type 3, Type 4
  NH=60 (Destination Options): Allow for Mobile IPv6

POLICY CHOICE:
  Unknown extension headers: Log and forward (RFC 7045 recommendation)
```

## ip6tables Extension Header Filtering

```bash
#!/bin/bash
# IPv6 extension header firewall rules

# Flush existing rules
sudo ip6tables -F FORWARD
sudo ip6tables -F INPUT
sudo ip6tables -F OUTPUT

# === Allow essential ICMPv6 (RFC 4890) ===
# Neighbor Discovery (NDP) must never be blocked
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 133 -j ACCEPT  # RS
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 134 -j ACCEPT  # RA
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 135 -j ACCEPT  # NS
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 136 -j ACCEPT  # NA
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 137 -j ACCEPT  # Redirect
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 2   -j ACCEPT  # Packet Too Big
sudo ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type 3   -j ACCEPT  # Time Exceeded

# === Fragment Header (NH=44) — ALLOW for legitimate fragmentation ===
sudo ip6tables -A FORWARD -m frag -j ACCEPT
sudo ip6tables -A INPUT   -m frag -j ACCEPT

# === IPsec Headers — ALLOW ===
sudo ip6tables -A FORWARD -p ah  -j ACCEPT
sudo ip6tables -A FORWARD -p esp -j ACCEPT
sudo ip6tables -A INPUT   -p ah  -j ACCEPT
sudo ip6tables -A INPUT   -p esp -j ACCEPT

# === Hop-by-Hop Options (NH=0) — ALLOW (required for MLD) ===
# Note: -m ipv6header matches extension headers
sudo ip6tables -A FORWARD -m ipv6header --header hop --soft -j ACCEPT
sudo ip6tables -A INPUT   -m ipv6header --header hop --soft -j ACCEPT

# === Routing Header Type 0 (RH0) — BLOCK (security risk) ===
# This is the deprecated amplification-attack-enabling routing header
sudo ip6tables -A FORWARD -m rt --rt-type 0 -j LOG --log-prefix "IPv6-RH0-DROP: "
sudo ip6tables -A FORWARD -m rt --rt-type 0 -j DROP
sudo ip6tables -A INPUT   -m rt --rt-type 0 -j DROP

# === Log unknown extension headers ===
# (Forward but log for monitoring - per RFC 7045)
sudo ip6tables -A FORWARD -m ipv6header --header dst --soft -j LOG --log-prefix "IPv6-DSTOPT: "
sudo ip6tables -A FORWARD -m ipv6header --header route --soft -j LOG --log-prefix "IPv6-ROUTE: "

echo "Extension header rules applied"
```

## nftables Equivalent

```
# /etc/nftables.conf
table inet filter6 {
    chain input {
        type filter hook input priority 0; policy drop;

        # Allow established/related
        ct state established,related accept

        # Essential ICMPv6 (RFC 4890 Section 4.4)
        ip6 nexthdr ipv6-icmp icmpv6 type {
            echo-request, echo-reply,
            nd-router-solicit, nd-router-advert,
            nd-neighbor-solicit, nd-neighbor-advert,
            nd-redirect, packet-too-big, time-exceeded,
            parameter-problem, mld-listener-query,
            mld-listener-report, mld-listener-done
        } accept

        # Fragment Header - allow
        ip6 nexthdr 44 accept

        # IPsec - allow
        ip6 nexthdr { 50, 51 } accept

        # Drop RH0 routing header
        ip6 nexthdr 43 ip6 rt type 0 drop

        # Log but allow other routing headers (MIPv6 type 2, SRH type 4)
        ip6 nexthdr 43 log prefix "IPv6-ROUTE: " accept

        # Allow Hop-by-Hop (required for MLD)
        ip6 nexthdr 0 accept

        # Allow TCP/UDP/ICMPv6
        ip6 nexthdr { 6, 17, 58 } accept

        # Log everything else
        log prefix "IPv6-DROP: " drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        # Similar rules for forwarded traffic
        ct state established,related accept
        ip6 nexthdr 44 accept
        ip6 nexthdr { 50, 51 } accept
        ip6 nexthdr 43 ip6 rt type 0 drop
        ip6 nexthdr { 0, 43, 60, 6, 17, 58 } accept
        log prefix "IPv6-FWD-DROP: " drop
    }
}
```

## Specific Extension Header Security Rules

```bash
# Additional security for Hop-by-Hop (prevent router exhaustion attacks)
# Only allow MLD router-alert (value=0) in Hop-by-Hop
# Block unusual/unknown Hop-by-Hop options that might exhaust router CPU
sudo ip6tables -A INPUT -m ipv6header --header hop --soft \
    -m u32 --u32 "48&0xff=0x05 && 50&0xffff=0x0000" \
    -j ACCEPT  # Allow: Router Alert option = 0 (MLD)

# Rate-limit Hop-by-Hop packets to prevent CPU exhaustion
sudo ip6tables -A INPUT -m ipv6header --header hop --soft \
    -m limit --limit 100/sec --limit-burst 200 \
    -j ACCEPT
sudo ip6tables -A INPUT -m ipv6header --header hop --soft \
    -j DROP  # Drop excess Hop-by-Hop packets
```

## Conclusion

Correct IPv6 extension header filtering requires explicitly allowing Fragment Headers (44) for fragmentation, IPsec headers (50/51), Hop-by-Hop (0) for MLD, and routing headers except Type 0 (which must be blocked). Unknown extension headers should generally be logged and forwarded per RFC 7045 rather than silently dropped. Overly aggressive filtering that blocks all extension headers breaks fragmentation, IPsec VPNs, and multicast — causing hard-to-diagnose connectivity failures.
