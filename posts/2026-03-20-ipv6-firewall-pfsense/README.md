# How to Configure IPv6 Firewall Rules on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, pfSense, Firewall, Stateful, FreeBSD

Description: Learn how to configure IPv6 firewall rules on pfSense, including interface rules, floating rules, ICMPv6 policy, and stateful connection tracking for IPv6 traffic.

## Overview

pfSense uses pf (Packet Filter) on FreeBSD for firewalling. IPv6 rules are configured in the same interface as IPv4 rules (Firewall → Rules), with an IPv6 address family selector. pfSense creates stateful rules by default, allowing return traffic for connections you initiate. This guide covers creating interface rules, floating rules, and IPv6-specific ICMPv6 policies.

## Prerequisites

- pfSense with a global IPv6 address on WAN
- IPv6 configured on LAN interface (SLAAC or DHCPv6)
- Verify IPv6 is active: **Interfaces → WAN → Check "Enable IPv6"**

## Interface Rules for IPv6

Navigate to **Firewall → Rules → LAN**:

### Allow IPv6 from LAN to WAN

```text
Rule 1: Allow IPv6 from LAN to any
  Action:           Pass
  Interface:        LAN
  Address Family:   IPv6
  Protocol:         Any
  Source:           LAN net (IPv6)
  Destination:      Any
  Description:      Allow outbound IPv6 from LAN
  State Type:       Keep State (default - stateful)
```

### Allow Only Specific Services Inbound

Navigate to **Firewall → Rules → WAN**:

```text
Rule 1: Allow HTTPS from anywhere (IPv6)
  Action:           Pass
  Interface:        WAN
  Address Family:   IPv6
  Protocol:         TCP
  Source:           Any
  Destination:      WAN Address (IPv6) / Port 443
  Description:      Allow inbound HTTPS IPv6

Rule 2: Allow SSH from management prefix only
  Action:           Pass
  Interface:        WAN
  Address Family:   IPv6
  Protocol:         TCP
  Source:           fd00:mgmt::/48 / Port 22
  Destination:      WAN Address (IPv6)
  Description:      SSH from management network only
```

## ICMPv6 Rules

pfSense has default ICMPv6 rules that allow essential types. Review them:

Navigate to **Firewall → Rules → WAN** and look for existing ICMPv6 rules:

```text
Essential ICMPv6 rules that should exist:
  - Pass ICMPv6 Destination Unreachable (type 1)
  - Pass ICMPv6 Packet Too Big (type 2) ← CRITICAL, never remove
  - Pass ICMPv6 Time Exceeded (type 3)
  - Pass ICMPv6 Parameter Problem (type 4)
  - Pass ICMPv6 Neighbor Solicitation/Advertisement (link-local only)
  - Pass ICMPv6 Router Advertisement (link-local only)
```

### Adding Custom ICMPv6 Rules

```text
Rule: Allow ping from specific prefix
  Action:           Pass
  Interface:        WAN
  Address Family:   IPv6
  Protocol:         ICMP
  ICMP Type:        Echo Request
  Source:           2001:db8:monitoring::/48
  Destination:      WAN Address (IPv6)
```

## Floating Rules (Apply to All Interfaces)

Navigate to **Firewall → Rules → Floating**:

```text
# Block Routing Header Type 0 on all interfaces

  Action:           Block
  Interface:        (all - leave blank for floating)
  Address Family:   IPv6
  Protocol:         TCP/UDP (won't block RH0 - need custom rule)
  Description:      Block deprecated RH0

# Note: pfSense doesn't have a direct RH0 block in the UI
# Use: Firewall → Rules → Advanced Options for custom pf rules
```

## CLI: Adding Custom pf Rules

For advanced IPv6 rules not available in the UI, SSH into pfSense:

```bash
# SSH to pfSense
ssh admin@pfSense.local

# View current IPv6 pf rules
pfctl -s rules | grep inet6

# Add temporary rule (lost on reboot)
pfctl -t blocklist6 -T add 2001:db8:bad::/48
pfctl -a 'user/ipv6-block' -f - << 'EOF'
table <ipv6-blocklist> { 2001:db8:bad::/48 }
block in inet6 from <ipv6-blocklist> to any label "IPv6 Blocklist"
EOF

# View current state table (conntrack equivalent)
pfctl -s states | grep inet6 | head -20
```

## State Table Management

pfSense's stateful firewall tracks IPv6 connections in the state table:

Navigate to **Diagnostics → States** and filter by:
- **IPv6** - Show only IPv6 states
- **Source** - Filter by specific IPv6 address

```bash
# CLI: View state table
pfctl -s states | grep 6   # IPv6 states

# Count states by source
pfctl -s states | grep inet6 | awk '{print $3}' | sort | uniq -c | sort -rn | head -20

# Flush all states (clears all connections)
pfctl -F states
```

## IPv6 Alias Groups

For maintainable firewall rules, create aliases:

Navigate to **Firewall → Aliases**:

```text
Name:             MGMT_IPv6
Type:             Network
Networks:
  fd00:mgmt::/48
  2001:db8:admin::/64
Description:      Management IPv6 networks

```

Then use in rules:
```text
Source: MGMT_IPv6
```

## Logging IPv6 Events

```bash
# Enable logging on firewall rules:
# Each rule has a "Log" checkbox in the UI

# View real-time firewall log:
# Status → System Logs → Firewall → Filter by IPv6

# From CLI:
clog /var/log/filter.log | grep IPv6 | tail -50
```

## Summary

pfSense IPv6 firewall rules are created under **Firewall → Rules** with **Address Family: IPv6** selected. The default state type (Keep State) makes rules stateful - return traffic is automatically allowed. Always verify that ICMPv6 Packet Too Big (type 2) is allowed on all interfaces. Use Aliases for managing groups of IPv6 prefixes. For advanced rules not available in the UI (like RH0 blocking), use SSH and custom pf rules. Monitor IPv6 connections via **Diagnostics → States** filtered by IPv6.
