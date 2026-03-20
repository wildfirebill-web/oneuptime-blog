# How to Troubleshoot IPv6 Firewall Blocking Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Firewall, ip6tables, nftables, Troubleshooting, Security, ICMPv6

Description: Diagnose and fix IPv6 traffic being blocked by firewalls, including ip6tables rules, nftables policies, and essential ICMPv6 exceptions required for IPv6 to function.

## Introduction

IPv6 firewalls can silently break IPv6 networking by blocking essential ICMPv6 messages. Unlike IPv4 where ICMP is optional, IPv6 relies on ICMPv6 for NDP, PMTUD, router discovery, and other critical functions. A firewall that blocks all ICMPv6 will appear to work (hosts can get addresses) but cause mysterious failures in connectivity and performance.

## Check Current ip6tables Rules

```bash
# Show all ip6tables rules
sudo ip6tables -L -n -v

# Check INPUT, OUTPUT, FORWARD chains
sudo ip6tables -L INPUT -n -v
sudo ip6tables -L OUTPUT -n -v
sudo ip6tables -L FORWARD -n -v

# Show rules with line numbers for editing
sudo ip6tables -L -n -v --line-numbers

# Check if default policy is DROP (blocks everything not explicitly allowed)
sudo ip6tables -L | grep "policy DROP"
```

## Check nftables Rules

```bash
# Show all nftables rules
sudo nft list ruleset

# Show specific table
sudo nft list table ip6 filter

# List chains
sudo nft list chains ip6

# Check default policy
sudo nft list chain ip6 filter input
```

## Essential ICMPv6 Rules (Must Allow)

```bash
# ICMPv6 types that MUST NOT be blocked:

# 1. Allow all ICMPv6 (simplest, recommended for basic setups)
sudo ip6tables -A INPUT -p icmpv6 -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 -j ACCEPT

# OR selectively allow critical types:

# NDP — Neighbor Discovery (required for on-link communication)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 135 -j ACCEPT  # NS
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 136 -j ACCEPT  # NA
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type 135 -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type 136 -j ACCEPT

# Router Discovery (required for SLAAC and default route)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 133 -j ACCEPT   # RS
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 134 -j ACCEPT   # RA

# PMTUD — Packet Too Big (required to avoid MTU black holes)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT

# Destination Unreachable and Time Exceeded (for connectivity feedback)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 1 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 3 -j ACCEPT
```

## Diagnosing Firewall as the Cause

```bash
# Temporarily flush all ip6tables rules to test
# WARNING: This removes ALL firewall protection
sudo ip6tables -F
sudo ip6tables -X
sudo ip6tables -P INPUT ACCEPT
sudo ip6tables -P OUTPUT ACCEPT
sudo ip6tables -P FORWARD ACCEPT

# Test if IPv6 works after flush
ping6 -c 3 2001:4860:4860::8888
curl -6 https://ipv6.google.com

# If it works after flush, a firewall rule was blocking it
# Add rules back carefully

# Check for packet drop counters
sudo ip6tables -L -n -v | grep -v "^$\|^Chain\|^target" | \
    awk '$1 > 0 {print}' | head -20
```

## Minimal Secure IPv6 ip6tables Ruleset

```bash
#!/bin/bash
# setup-ipv6-firewall.sh

# Flush existing rules
ip6tables -F
ip6tables -X

# Default policies
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT ACCEPT

# Allow established connections
ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
ip6tables -A INPUT -i lo -j ACCEPT

# Allow ICMPv6 (essential for IPv6 to function)
ip6tables -A INPUT -p icmpv6 -j ACCEPT
ip6tables -A FORWARD -p icmpv6 -j ACCEPT

# Allow SSH
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

echo "IPv6 firewall configured"
ip6tables -L -n -v
```

## nftables Equivalent

```bash
# /etc/nftables.conf — IPv6 firewall with ICMPv6 allowed
cat << 'EOF' | sudo tee /etc/nftables.conf
table ip6 filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif lo accept
        ct state established,related accept
        ip6 nexthdr icmpv6 accept  # Allow all ICMPv6
        tcp dport { ssh, http, https } accept
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
    chain forward {
        type filter hook forward priority 0; policy drop;
        ip6 nexthdr icmpv6 accept  # Allow ICMPv6 forwarding
    }
}
EOF
sudo systemctl restart nftables
```

## Conclusion

IPv6 firewalls must allow ICMPv6 to function correctly. The most common mistake is blocking all ICMP including ICMPv6, which breaks NDP (Layer 2 resolution), PMTUD (MTU discovery), and router/prefix discovery. Always allow ICMPv6 types 1, 2, 3, 133, 134, 135, and 136 at minimum. Test by temporarily flushing ip6tables rules — if IPv6 immediately works, a rule is the cause. Use packet drop counters (`ip6tables -L -n -v`) to identify which rule is dropping traffic.
