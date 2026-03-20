# How to Configure IPv6 Firewall Rules on OPNsense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, OPNsense, Firewall, FreeBSD, Pf

Description: Learn how to configure IPv6 firewall policies on OPNsense, covering interface rules, floating rules, ICMPv6 settings, and monitoring IPv6 traffic flows.

## Overview

OPNsense is a FreeBSD-based firewall using pf, similar to pfSense but with a redesigned UI and focus on security. IPv6 firewalling is integrated throughout the rule configuration interface. Rules can be IPv6-only, IPv4-only, or both. OPNsense includes ICMPv6 neighbor discovery passthrough by default.

## Enabling IPv6 on Interfaces

Navigate to **Interfaces → WAN**:
- Enable: **IPv6 Configuration Type** → DHCPv6 or Static
- Note the assigned IPv6 address

Navigate to **Interfaces → LAN**:
- IPv6 Configuration Type: Track Interface or Static
- Track WAN: Select WAN, prefix delegation size /48

## Creating IPv6 Firewall Rules

Navigate to **Firewall → Rules → LAN**:

### Allow IPv6 Outbound from LAN

```text
Action:           Pass
Interface:        LAN
Direction:        in
TCP/IP Version:   IPv6
Protocol:         Any
Source:           LAN net (IPv6)
Destination:      Any
Allow options:    ✓ (enabled by default - stateful)
Description:      Allow all IPv6 from LAN

```

### WAN Inbound Rules

Navigate to **Firewall → Rules → WAN**:

```text
# Allow HTTPS

Action:           Pass
Interface:        WAN
TCP/IP Version:   IPv6
Protocol:         TCP
Source:           Any
Destination:      WAN address
Destination port: HTTPS (443)
Description:      Allow inbound HTTPS IPv6

# Allow SSH from management only
Action:           Pass
Interface:        WAN
TCP/IP Version:   IPv6
Protocol:         TCP
Source:           fd00:mgmt::/48
Destination:      WAN address
Destination port: SSH (22)
Description:      Management SSH IPv6
```

## ICMPv6 Configuration

OPNsense automatically handles essential ICMPv6. Verify under **Firewall → Rules → Floating**:

```text
# Automatic rules include:
# Allow Neighbor Discovery (Solicitation/Advertisement)
# Allow Router Advertisement/Solicitation
# Allow Packet Too Big (required for PMTUD)
```

### Custom ICMPv6 Rules

```text
# Allow ping from monitoring network
Action:           Pass
Interface:        WAN
TCP/IP Version:   IPv6
Protocol:         ICMP
ICMP type:        Echo Request (128)
Source:           2001:db8:monitor::/48
Destination:      WAN address
Description:      Allow IPv6 ping from monitoring
```

## Floating Rules (Multi-Interface)

Navigate to **Firewall → Rules → Floating**:

```text
# Block traffic from bogon IPv6 sources on all interfaces
Action:           Block
Interface:        (all - checked boxes for all interfaces)
Direction:        in
TCP/IP Version:   IPv6
Source:           2001:db8::/32 (Documentation prefix)
Description:      Block IPv6 documentation prefix
```

## Aliases for IPv6 Prefixes

Navigate to **Firewall → Aliases → Add**:

```text
Name:             IPv6_Management
Type:             Network
Network(s):
  fd00:mgmt::/48
  2001:db8:ops::/64
Description:      IPv6 Management Networks
```

Use in rules as source/destination.

## State Table (Connection Tracking)

Navigate to **Firewall → Diagnostics → States**:

```text
# Filter options:
Protocol:   IPv6
Interface:  WAN

# Shows active IPv6 connections:
# State   Proto  Source                Destination          Flags
# ESTAB   TCP    2001:db8:client::1:54321 2001:db8:server::1:443
```

```bash
# From OPNsense shell:
ssh root@opnsense.local

# View IPv6 state table
pfctl -s states | grep inet6

# Count states
pfctl -s states | grep inet6 | wc -l
```

## Logging IPv6 Firewall Events

```text
# Enable logging on a rule:
# In rule editor: Logging → Enable ✓

# View logs:
# Reporting → System Health → Firewall
# Filter: IPv6

# From CLI:
clog /var/log/filterlog.log | grep ',6,' | tail -50
# CSV format: time, rule, direction, interface, proto=6 (IPv6)
```

## Key Differences: OPNsense vs pfSense IPv6 Firewall

| Feature | OPNsense | pfSense |
|---------|----------|---------|
| ICMPv6 auto rules | Built-in, visible in Floating | Built-in, less visible |
| UI for IPv6 | IPv6 column in rules list | TCP/IP version field |
| Plugin security | OPNsense has security audit checks | Package-based |
| Logging | filterlog CSV format | Similar CSV format |

## Verify IPv6 Connectivity Through Firewall

```bash
# From behind OPNsense LAN:
ping6 -c 3 2001:4860:4860::8888   # Google IPv6 DNS

# Check firewall isn't blocking PMTUD
ping6 -c 3 -s 1400 2001:4860:4860::8888   # Large packet

# Traceroute6 to verify path
traceroute6 ipv6.google.com
```

## Summary

OPNsense IPv6 firewall rules are configured under **Firewall → Rules** with **TCP/IP Version: IPv6**. Default auto-rules handle essential ICMPv6 including Neighbor Discovery and Packet Too Big. Create Aliases for common IPv6 prefixes (management networks, trusted sources) and reference them in rules for maintainability. Use Floating rules for policies that apply across all interfaces. Monitor active IPv6 connections under **Firewall → Diagnostics → States** filtered to IPv6. Enable logging per-rule for audit trail.
