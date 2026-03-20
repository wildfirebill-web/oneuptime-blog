# How to Configure ip6tables Basic Rules for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ip6tables, Firewall, Linux, Security

Description: Learn the fundamentals of ip6tables for IPv6 firewalling on Linux, including tables, chains, rule syntax, and building a basic but complete IPv6 firewall policy.

## Overview

ip6tables is the Linux kernel's IPv6 packet filtering framework - the IPv6 counterpart to iptables. It uses the same netfilter architecture with tables (filter, mangle, nat) and chains (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING). ip6tables rules control which IPv6 packets are accepted, dropped, or modified.

## ip6tables vs iptables

| Aspect | iptables | ip6tables |
|--------|----------|-----------|
| Protocol | IPv4 | IPv6 |
| NAT support | Full | Limited (no MASQUERADE in older kernels) |
| ICMPv6 module | icmp | icmpv6 |
| Config file | /etc/iptables/rules.v4 | /etc/iptables/rules.v6 |
| Status command | iptables -L -n | ip6tables -L -n |

## ip6tables Rule Syntax

```bash
ip6tables -[ADIRF] chain [-s src] [-d dst] [-p proto] [-i in-iface] [-o out-iface] \
          [protocol-specific options] -j TARGET

# Common flags:

# -A: Append rule to chain
# -I: Insert rule at position
# -D: Delete rule
# -F: Flush (delete all rules in chain)
# -P: Set default policy
# -L: List rules
# -n: Numeric output (don't resolve hostnames)
# -v: Verbose (show counters)
# --line-numbers: Show rule numbers

# Common targets:
# ACCEPT: Allow the packet
# DROP:   Silently discard
# REJECT: Discard + send error
# LOG:    Log to syslog
# RETURN: Return from chain
```

## Building a Basic IPv6 Firewall

### Step 1: Set Default Policies

```bash
# Set restrictive defaults
ip6tables -P INPUT   DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT  ACCEPT   # Allow all outbound by default
```

### Step 2: Allow Loopback

```bash
# Allow all traffic on the loopback interface
ip6tables -A INPUT  -i lo -j ACCEPT
ip6tables -A OUTPUT -o lo -j ACCEPT
```

### Step 3: Allow Established Connections

```bash
# Allow replies to connections we initiated
ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Or use conntrack module (newer)
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Step 4: Allow Essential ICMPv6 (RFC 4890)

```bash
# Packet Too Big - MUST allow (required for PMTUD)
ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Destination Unreachable
ip6tables -A INPUT -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT

# Time Exceeded (traceroute)
ip6tables -A INPUT -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT

# Parameter Problem
ip6tables -A INPUT -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT

# NDP - only from link-local sources
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type router-solicitation -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-solicitation -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-advertisement -j ACCEPT

# Echo (ping) - allow but rate-limit
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -m limit --limit 5/s -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-reply -j ACCEPT
```

### Step 5: Allow Specific Services

```bash
# SSH from management network
ip6tables -A INPUT -p tcp --dport 22 -s fd00:mgmt::/48 -j ACCEPT

# Web server
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
```

### Step 6: Drop Bogon Sources

```bash
ip6tables -A INPUT -s ::/128 -j DROP          # Unspecified
ip6tables -A INPUT -s ::1/128 -j DROP         # Loopback (except lo already handled)
ip6tables -A INPUT -s 2001:db8::/32 -j DROP   # Documentation prefix
ip6tables -A INPUT -s fc00::/7 -j DROP        # ULA from internet
```

### Step 7: Logging Before Final Drop

```bash
# Log dropped packets for analysis
ip6tables -A INPUT -j LOG --log-prefix "IPv6-INPUT-DROP: " --log-level 4
```

## Viewing Current Rules

```bash
# List all rules with counters
ip6tables -L -n -v

# List with line numbers (for deletion)
ip6tables -L -n --line-numbers

# List specific chain
ip6tables -L INPUT -n -v

# List specific table
ip6tables -t mangle -L -n
```

## Deleting Rules

```bash
# Delete by line number
ip6tables -D INPUT 3

# Delete by specification (must match exactly)
ip6tables -D INPUT -p tcp --dport 22 -j ACCEPT

# Flush entire chain
ip6tables -F INPUT
```

## Complete Example: Server Firewall

```bash
#!/bin/bash
# /usr/local/sbin/ip6tables-server.sh

ip6tables -F   # Flush all chains
ip6tables -X   # Delete user-defined chains

# Default policies
ip6tables -P INPUT   DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT  ACCEPT

# Loopback
ip6tables -A INPUT -i lo -j ACCEPT

# Established
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Critical ICMPv6
ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 -j ACCEPT

# Services
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# Log drops
ip6tables -A INPUT -j LOG --log-prefix "IPv6-DROP: "
```

## Summary

ip6tables uses the same syntax as iptables with the key difference of the `--icmpv6-type` match (instead of `--icmp-type`). Always set DROP as the default INPUT policy, allow loopback, allow ESTABLISHED/RELATED connections, and explicitly permit required ICMPv6 types (especially packet-too-big). Use `ip6tables -L -n -v` to verify rules. Save rules with `ip6tables-save > /etc/iptables/rules.v6` and restore with `ip6tables-restore < /etc/iptables/rules.v6`.
