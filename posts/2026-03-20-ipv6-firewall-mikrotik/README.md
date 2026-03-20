# How to Configure IPv6 Firewall Rules on MikroTik RouterOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MikroTik, RouterOS, Firewall, ip6tables

Description: Learn how to configure IPv6 firewall rules on MikroTik RouterOS using the /ipv6 firewall filter commands, including input, forward, and output chain rules.

## Overview

MikroTik RouterOS has a dedicated IPv6 firewall filter (`/ipv6 firewall filter`) separate from IPv4 firewalling. It supports stateful connection tracking, ICMPv6 rules, source/destination prefix matching, and protocol-specific rules. The IPv6 firewall is critical to configure since MikroTik forwards IPv6 to internal hosts when they have global addresses from prefix delegation.

## Checking Current IPv6 Firewall

```bash
# View current IPv6 firewall rules
/ipv6 firewall filter print

# View connection tracking
/ipv6 firewall connection print

# View IPv6 addresses
/ipv6 address print
```

## Basic IPv6 Input Chain Rules

```bash
# Add rules to INPUT chain (protect the router itself)

# Allow established and related connections
/ipv6 firewall filter add chain=input connection-state=established,related action=accept comment="Allow established/related"

# Drop invalid packets
/ipv6 firewall filter add chain=input connection-state=invalid action=drop comment="Drop invalid"

# Allow loopback
/ipv6 firewall filter add chain=input in-interface=lo action=accept comment="Allow loopback"

# Allow ICMPv6 essential types
# Destination Unreachable
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=1:0-255 action=accept comment="ICMPv6 Unreachable"

# Packet Too Big — NEVER block
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=2:0-255 action=accept comment="ICMPv6 Packet Too Big"

# Time Exceeded
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=3:0-255 action=accept comment="ICMPv6 Time Exceeded"

# Parameter Problem
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=4:0-255 action=accept comment="ICMPv6 Parameter Problem"

# NDP — Router and Neighbor messages (link-local only)
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=133:0-255 src-address=fe80::/10 action=accept comment="Router Solicitation"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=134:0-255 src-address=fe80::/10 action=accept comment="Router Advertisement"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=135:0-255 src-address=fe80::/10 action=accept comment="Neighbor Solicitation"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=136:0-255 src-address=fe80::/10 action=accept comment="Neighbor Advertisement"

# Allow SSH from management network
/ipv6 firewall filter add chain=input protocol=tcp dst-port=22 src-address=fd00:mgmt::/48 action=accept comment="SSH from management"

# Drop everything else to router
/ipv6 firewall filter add chain=input action=log log-prefix="IPv6-INPUT-DROP: "
/ipv6 firewall filter add chain=input action=drop comment="Drop all other input"
```

## Forward Chain Rules

```bash
# Allow forwarded traffic to established connections
/ipv6 firewall filter add chain=forward connection-state=established,related action=accept comment="Allow established forward"

# Drop invalid forwarded packets
/ipv6 firewall filter add chain=forward connection-state=invalid action=drop comment="Drop invalid forward"

# Allow Packet Too Big through (required for PMTUD)
/ipv6 firewall filter add chain=forward protocol=icmpv6 icmp-options=2:0-255 action=accept comment="PTB forward"

# Allow new connections from internal → external
/ipv6 firewall filter add chain=forward in-interface=bridge1 out-interface=ether1-wan action=accept comment="LAN to WAN IPv6"

# Drop new connections from external → internal (implicit deny)
/ipv6 firewall filter add chain=forward in-interface=ether1-wan out-interface=bridge1 connection-state=new action=drop comment="Block new inbound"

# Log drops
/ipv6 firewall filter add chain=forward action=log log-prefix="IPv6-FWD-DROP: "
/ipv6 firewall filter add chain=forward action=drop
```

## Blocking Specific Prefixes

```bash
# Block specific attacker prefix
/ipv6 firewall filter add chain=input src-address=2001:db8:bad::/48 action=drop comment="Block attacker"

# Block bogon sources at input
/ipv6 firewall filter add chain=input src-address=2001:db8::/32 action=drop comment="Block documentation prefix"
/ipv6 firewall filter add chain=input src-address=::/128 action=drop comment="Block unspecified"
```

## Address Lists (MikroTik Equivalent of ipset)

```bash
# Create address list for management
/ipv6 firewall address-list add list=mgmt address=fd00:mgmt::/48 comment="Management network"
/ipv6 firewall address-list add list=mgmt address=2001:db8:admin::1/128 comment="Admin workstation"

# Use in firewall rule
/ipv6 firewall filter add chain=input protocol=tcp dst-port=22 src-address-list=mgmt action=accept comment="SSH from address list"

# Block an address list
/ipv6 firewall address-list add list=blocklist6 address=2001:db8:bad::/48
/ipv6 firewall filter add chain=input src-address-list=blocklist6 action=drop comment="Drop blocklist"
```

## Rate Limiting IPv6 (Connection Limit)

```bash
# Limit SSH connections per source
/ipv6 firewall filter add chain=input protocol=tcp dst-port=22 connection-limit=5,32 action=drop comment="SSH connection limit per /128"

# Rate limit new connections using connection-rate
/ipv6 firewall filter add chain=input protocol=tcp dst-port=80 connection-state=new connection-rate=100/1s action=drop comment="HTTP rate limit"
```

## Viewing and Managing Rules

```bash
# List rules with numbers
/ipv6 firewall filter print

# Remove a rule by number
/ipv6 firewall filter remove numbers=5

# Enable/disable a rule
/ipv6 firewall filter set numbers=5 disabled=yes
/ipv6 firewall filter set numbers=5 disabled=no

# Move rule to different position
/ipv6 firewall filter move numbers=5 destination=2
```

## Summary

MikroTik RouterOS IPv6 firewall is configured under `/ipv6 firewall filter` with separate INPUT (router protection) and FORWARD (pass-through traffic) chains. Add rules in order: established/related accept, invalid drop, essential ICMPv6 accept (types 1-4 and NDP 133-137 from link-local only), service-specific accept, then drop. Use address lists (`/ipv6 firewall address-list`) for maintainable prefix groups. The forward chain allows IPv6 from LAN to WAN but blocks new connections from WAN to LAN. Always place LOG rules before DROP rules for troubleshooting.
