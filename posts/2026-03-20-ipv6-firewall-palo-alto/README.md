# How to Configure IPv6 Firewall Rules on Palo Alto Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Palo Alto, Firewall, NGFW, Security Policy

Description: Learn how to configure IPv6 security policies on Palo Alto Networks next-generation firewalls, including address objects, security rules, threat profiles, and App-ID for IPv6.

## Overview

Palo Alto Networks NGFW (PAN-OS) supports IPv6 in its security policy framework. IPv6 addresses and prefixes are defined as address objects and used in security rules the same way as IPv4. PAN-OS supports App-ID inspection for IPv6 traffic and applies all security profiles (threat prevention, URL filtering) to IPv6 flows.

## Enable IPv6 on Interfaces

Navigate to **Network → Interfaces → Ethernet → Edit**:

```text
IPv6 tab:
  Enable IPv6 on Interface: ✓
  Address: 2001:db8:wan::1/64
  Enable Neighbor Discovery: ✓
```

Or via CLI:

```bash
# PAN-OS CLI

set network interface ethernet ethernet1/1 ipv6 enabled yes
set network interface ethernet ethernet1/1 ipv6 prefix 2001:db8:wan::1/64 enable-on-interface yes
set network interface ethernet ethernet1/2 ipv6 enabled yes
set network interface ethernet ethernet1/2 ipv6 prefix 2001:db8:lan::1/64
```

## IPv6 Address Objects

Navigate to **Objects → Addresses → Add**:

```text
Name:       IPv6-INTERNAL
Type:       IP Netmask
Address:    2001:db8:lan::/48 (IPv6 prefix)
```

```bash
# CLI
set address IPv6-INTERNAL ip-netmask 2001:db8:lan::/48
set address IPv6-MGMT ip-netmask fd00:mgmt::/48
set address IPv6-WEB-SERVER ip-netmask 2001:db8:lan::web/128
```

## Security Policies for IPv6

Navigate to **Policies → Security → Add**:

### Allow Outbound IPv6

```text
Name:           Allow-Outbound-IPv6
Source Zone:    Trust
Destination:    Untrust
Source Address: IPv6-INTERNAL
Dest Address:   any
Application:    any
Service:        application-default
Action:         Allow
Profile Group:  Best-Practice   (applies threat prevention)
Log:            ✓ Log at Session End
```

### Allow Specific Inbound Services

```text
Name:           Allow-HTTPS-IPv6
Source Zone:    Untrust
Destination:    DMZ
Source Address: any
Dest Address:   IPv6-WEB-SERVER
Application:    ssl (App-ID detects HTTPS)
Service:        application-default
Action:         Allow
Profile Group:  Best-Practice
```

### SSH from Management Only

```text
Name:           Allow-SSH-Mgmt-IPv6
Source Zone:    Untrust
Destination:    Trust
Source Address: IPv6-MGMT
Dest Address:   any
Application:    ssh
Service:        application-default
Action:         Allow
```

## ICMPv6 in PAN-OS

PAN-OS handles ICMPv6 Neighbor Discovery automatically for configured interfaces. For security policies:

```bash
# Create service object for ICMPv6 PTB
set service ICMPv6-PTB protocol tcp port 0
# Note: Use application objects for ICMP in PAN-OS

# PAN-OS App-ID covers icmp for both IPv4 and IPv6 pings
# For critical ICMPv6 types, use a separate security rule:

# Add rule allowing essential ICMPv6
set rulebase security rules ICMPv6-Essential from any
set rulebase security rules ICMPv6-Essential to any
set rulebase security rules ICMPv6-Essential source any
set rulebase security rules ICMPv6-Essential destination any
set rulebase security rules ICMPv6-Essential application icmp6
set rulebase security rules ICMPv6-Essential service application-default
set rulebase security rules ICMPv6-Essential action allow
```

## Threat Prevention for IPv6

PAN-OS applies the same threat prevention profiles to IPv6:

```text
Profile Group: Best-Practice
  - Antivirus: ✓
  - Anti-Spyware: ✓ (includes IPS)
  - Vulnerability Protection: ✓
  - URL Filtering: ✓ (works for IPv6 connections to web)
  - File Blocking: ✓
  - WildFire: ✓
```

These profiles inspect IPv6 application payloads identically to IPv4.

## Monitoring IPv6 Traffic

```bash
# CLI: View session table filtered to IPv6
show session all filter protocol 6 ipv6 yes

# Monitor active IPv6 sessions
debug dataplane state | match ipv6

# View traffic log for IPv6 (GUI: Monitor → Logs → Traffic)
# Filter: addr.src in fd00::/8 or addr.src in 2001:db8::/32
```

Navigate to **Monitor → Logs → Traffic** and filter:
```text
addr.src in 2001:db8::/32 and ( app eq ssl or app eq ssh )
```

## IPv6 Address Groups

```bash
# Create address group
set address-group IPv6-TRUSTED member IPv6-INTERNAL
set address-group IPv6-TRUSTED member IPv6-MGMT
```

## Commit and Verify

```bash
# PAN-OS always requires commit
commit description "Add IPv6 firewall policies"

# Verify interfaces
show interface ethernet1/1 | match ipv6

# Verify security rules are applied
show running security-policy name Allow-Outbound-IPv6

# Test with packet-diag
debug dataplane packet-diag set filter match ipv6 source 2001:db8:ext::1 destination 2001:db8:lan::web sport 0 dport 443
debug dataplane packet-diag set filter on
```

## Summary

Palo Alto PAN-OS IPv6 security policies use the same framework as IPv4. Create address objects with IPv6 prefixes (`ip-netmask 2001:db8::/48`) and reference them in security rules with source/destination address. App-ID works for IPv6 - use application names (`ssl`, `ssh`, `icmp6`) rather than port numbers. Apply the same threat prevention profile groups to IPv6 policies as IPv4. All changes require `commit`. Monitor IPv6 traffic via **Monitor → Logs → Traffic** with IPv6 address filters.
