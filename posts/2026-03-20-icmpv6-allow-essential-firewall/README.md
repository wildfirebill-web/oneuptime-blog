# How to Allow Essential ICMPv6 Through a Firewall

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Firewall, IPv6 Security, ip6tables, nftables

Description: Configure firewall rules to allow essential ICMPv6 messages while blocking non-essential ones, using ip6tables, nftables, and firewalld with practical rule examples.

## Introduction

The minimal ICMPv6 allow-list for a working IPv6 network requires allowing specific message types based on whether you're configuring a host firewall or a transit firewall. This guide provides ready-to-use firewall rules for the most common tools (ip6tables, nftables, firewalld) that ensure IPv6 connectivity is maintained while limiting unnecessary ICMPv6 exposure.

## Minimum ICMPv6 Rules for a Host

```bash
# Host firewall minimum: allows IPv6 to function completely

# Using ip6tables — flush existing ICMPv6 rules first if needed
# sudo ip6tables -F

# 1. Packet Too Big (PMTUD — CRITICAL)
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# 2. Error messages (important for connectivity diagnosis)
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT

# 3. NDP (CRITICAL — without these, IPv6 doesn't work at all)
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type router-advertisement   -j ACCEPT
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type neighbor-solicitation   -j ACCEPT
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type neighbor-advertisement  -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type router-solicitation     -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type neighbor-solicitation   -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type neighbor-advertisement  -j ACCEPT

# 4. MLD (for multicast; required for SLAAC to work)
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type 130 -j ACCEPT  # MLD Query
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type 131 -j ACCEPT  # MLD Report
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type 132 -j ACCEPT  # MLD Done
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type 143 -j ACCEPT  # MLDv2 Report
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type 131 -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type 143 -j ACCEPT

# 5. Echo (useful for diagnostics, optional but recommended)
sudo ip6tables -A INPUT  -p icmpv6 --icmpv6-type echo-request -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type echo-reply   -j ACCEPT
```

## Transit Router/Firewall Rules

```bash
# Transit firewall: allow ICMPv6 through but block link-local-only types

# Allow essential ICMPv6 through (FORWARD chain)
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type packet-too-big         -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type time-exceeded           -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type parameter-problem       -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type echo-request            -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type echo-reply              -j ACCEPT

# Block NDP at transit (link-local only — should never be forwarded)
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type router-solicitation    -j DROP
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type router-advertisement   -j DROP
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type neighbor-solicitation  -j DROP
sudo ip6tables -A FORWARD -p icmpv6 -m icmpv6 --icmpv6-type neighbor-advertisement -j DROP
```

## nftables Equivalent

```bash
# Complete nftables configuration for a host with essential ICMPv6

sudo nft -f - << 'EOF'
table ip6 filter {
    chain input {
        type filter hook input priority 0;

        # Allow established/related traffic
        ct state established,related accept
        ct state invalid drop

        # Essential ICMPv6 (NEVER block these)
        icmpv6 type packet-too-big accept
        icmpv6 type { destination-unreachable, time-exceeded, parameter-problem } accept

        # NDP (required for IPv6 address resolution and SLAAC)
        icmpv6 type { nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert } accept

        # MLD (required for multicast)
        icmpv6 type { mld-listener-query, mld-listener-report, mld-listener-done } accept
        icmpv6 type mld2-listener-report accept

        # Echo (optional but useful)
        icmpv6 type echo-request accept

        # Accept loopback
        iif lo accept
    }
}
EOF
```

## firewalld Configuration

```bash
# Using firewalld: allow essential ICMPv6
# firewalld allows ICMPv6 by default; verify:
sudo firewall-cmd --query-icmp-block-inversion

# List blocked ICMPv6 types
sudo firewall-cmd --list-icmp-blocks

# Ensure packet-too-big is NOT blocked
sudo firewall-cmd --remove-icmp-block=packet-too-big --permanent
sudo firewall-cmd --reload

# If firewalld is blocking all ICMPv6, remove the block:
sudo firewall-cmd --remove-icmp-block=neighbour-advertisement --permanent
sudo firewall-cmd --remove-icmp-block=neighbour-solicitation --permanent
sudo firewall-cmd --reload

# Check final state
sudo firewall-cmd --list-all
```

## Conclusion

Allowing essential ICMPv6 requires a specific allow-list rather than open acceptance of all ICMPv6. For hosts: allow Packet Too Big, NDP types (RA/NS/NA), MLD, and error messages. For transit firewalls: allow the same error messages and Echo, but block NDP at the transit boundary. Using named icmpv6-type values in ip6tables (like `--icmpv6-type packet-too-big`) is more readable and maintainable than using numeric type values. The nftables approach with sets of icmpv6 types is the most concise modern option.
