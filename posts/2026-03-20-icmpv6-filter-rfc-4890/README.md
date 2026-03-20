# How to Filter ICMPv6 Following RFC 4890

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, RFC 4890, Firewall, IPv6 Security, Filtering

Description: Implement ICMPv6 filtering policies following RFC 4890, understanding which messages to allow, rate-limit, or drop at different network boundary points.

## Introduction

RFC 4890 ("Recommendations for Filtering ICMPv6 Messages in Firewalls") provides a comprehensive framework for ICMPv6 filtering. It distinguishes between different boundary types (transit firewalls, host firewalls, customer edge routers) and provides specific recommendations for each. Unlike IPv4 ICMP filtering, IPv6 requires a carefully considered allow-list approach rather than a deny-all policy.

## RFC 4890 Filtering Categories

```text
RFC 4890 classifies ICMPv6 messages into:

Category 1: SHOULD NOT be filtered (essential for IPv6 operation)
  Type 1   - Destination Unreachable
  Type 2   - Packet Too Big (CRITICAL: never filter)
  Type 3   - Time Exceeded
  Type 4   - Parameter Problem
  Type 128 - Echo Request
  Type 129 - Echo Reply

Category 2: SHOULD NOT be filtered on local links
  Type 133 - Router Solicitation
  Type 134 - Router Advertisement
  Type 135 - Neighbor Solicitation
  Type 136 - Neighbor Advertisement
  Type 137 - Redirect

Category 3: SHOULD be filtered at transit firewalls
  Type 133-137 - NDP (valid only on-link; routing them = configuration error)

Category 4: MAY be filtered (use case dependent)
  Type 130-132 - MLD (multicast; filter if multicast not used)
  Type 143 - MLDv2 (same)

Category 5: SHOULD be filtered (dangerous)
  Type 0  - Unassigned (reserved, should not exist in traffic)
  Type 149 - SEND Certificate Path Advertisement (unless using SEND)
```

## Transit Firewall Policy (RFC 4890 Compliant)

```bash
# Transit firewall: sits between the Internet and your network

# Allows routable IPv6 traffic + essential ICMPv6

# Start with a clean slate for ICMPv6
sudo ip6tables -F INPUT
sudo ip6tables -F OUTPUT
sudo ip6tables -F FORWARD

# RFC 4890 Section 4.3.1: Error messages to allow through
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 1   -j ACCEPT  # Dest Unreachable
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 2   -j ACCEPT  # Packet Too Big
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 3   -j ACCEPT  # Time Exceeded
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 4   -j ACCEPT  # Parameter Problem

# Echo (diagnostic - allow in both directions)
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 128 -j ACCEPT  # Echo Request
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 129 -j ACCEPT  # Echo Reply

# RFC 4890 Section 4.3.1: NDP must NOT be forwarded across transit
# Block NDP at transit boundaries (valid only on-link)
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 133 -j DROP  # RS
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 134 -j DROP  # RA
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 135 -j DROP  # NS
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 136 -j DROP  # NA
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type 137 -j DROP  # Redirect

# Drop unassigned ICMPv6 types (RFC 4890 Section 4.3.3)
# Everything else is dropped by the default policy
```

## Local Segment Policy (Host Firewall)

```bash
# Host firewall: allows local network ICMPv6

# Allow all essential error messages
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 1   -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2   -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 3   -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 4   -j ACCEPT

# Allow NDP on local segment
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 133 -j ACCEPT  # RS
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 -j ACCEPT  # RA
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 -j ACCEPT  # NS
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 136 -j ACCEPT  # NA

# Allow MLD (for IPv6 multicast to work)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 130 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 131 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 132 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 143 -j ACCEPT

# Allow ping6
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type 128 -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type 129 -j ACCEPT
```

## nftables Implementation

```bash
# nftables version of RFC 4890-compliant filtering
sudo nft add table ip6 filter
sudo nft add chain ip6 filter input  '{ type filter hook input priority 0; policy drop; }'
sudo nft add chain ip6 filter forward '{ type filter hook forward priority 0; policy drop; }'

# Essential ICMPv6 (never block these)
sudo nft add rule ip6 filter input icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem } accept
sudo nft add rule ip6 filter forward icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem } accept

# NDP - allow on input, block on forward
sudo nft add rule ip6 filter input  icmpv6 type { nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, nd-redirect } accept
sudo nft add rule ip6 filter forward icmpv6 type { nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } drop

# Echo
sudo nft add rule ip6 filter input icmpv6 type echo-request accept
```

## Conclusion

RFC 4890 provides a clear and practical framework for ICMPv6 filtering. The key principles: allow all four error message types (Types 1-4) at all boundary types; block NDP (Types 133-137) at transit firewalls but allow them on local segments; allow MLD (Types 130-132, 143) on segments using multicast. The distinction between "local link" and "transit" boundaries is the most important concept - NDP messages are only valid on the link where they originate and must never be routed.
