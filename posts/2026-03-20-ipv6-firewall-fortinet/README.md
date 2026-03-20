# How to Configure IPv6 Firewall Policies on Fortinet FortiGate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, FortiGate, Firewall, Fortinet, Security Policy

Description: Learn how to configure IPv6 firewall policies on Fortinet FortiGate, including address objects, IPv6 policy rules, inspection profiles, and logging.

## Overview

Fortinet FortiGate supports IPv6 in its next-generation firewall policies. IPv6 is configured using the same policy framework as IPv4 — you create IPv6 address objects and firewall policies with IPv6 address family. FortiGate performs stateful inspection and deep packet inspection for IPv6 traffic.

## Enable IPv6 on Interfaces

```bash
# FortiOS CLI: Enable IPv6 on interfaces
config system interface
    edit "wan1"
        set ip6 2001:db8:wan::1/64
        set ipv6-allow-any-host enable
        next
    edit "lan"
        set ip6 2001:db8:lan::1/64
        next
end
```

## IPv6 Address Objects

```bash
# Create IPv6 address objects
config firewall address6
    edit "INTERNAL-NET-V6"
        set type ipprefix
        set ip6 2001:db8:lan::/48
        next
    edit "MGMT-NET-V6"
        set type ipprefix
        set ip6 fd00:mgmt::/48
        next
    edit "WEB-SERVER-V6"
        set type ipprefix
        set ip6 2001:db8:lan::web/128
        next
    edit "ANY-V6"
        set type ipprefix
        set ip6 ::/0
        next
end

# Create address group
config firewall addrgrp6
    edit "TRUSTED-V6"
        set member "INTERNAL-NET-V6" "MGMT-NET-V6"
        next
end
```

## IPv6 Firewall Policies (GUI Path)

Navigate to **Policy & Objects → Firewall Policy → Create New → IPv6 Policy**:

```
Name:             Allow-Outbound-V6
From:             LAN
To:               WAN
IPv6 Source:      INTERNAL-NET-V6
IPv6 Destination: ANY-V6
Service:          ALL
Action:           ACCEPT
Inspection Mode:  Flow-based
Log Traffic:      All Sessions
NAT:              Disable (IPv6 rarely uses NAT)
```

## IPv6 Policies via CLI

```bash
# Outbound IPv6 policy
config firewall policy6
    edit 100
        set name "Allow-Outbound-IPv6"
        set srcintf "lan"
        set dstintf "wan1"
        set srcaddr "INTERNAL-NET-V6"
        set dstaddr "ALL_IPv6"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
        set comments "Allow outbound IPv6 from LAN"
        next
    edit 101
        set name "Allow-HTTPS-Inbound"
        set srcintf "wan1"
        set dstintf "lan"
        set srcaddr "ALL_IPv6"
        set dstaddr "WEB-SERVER-V6"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set logtraffic all
        next
    edit 102
        set name "Allow-SSH-Management"
        set srcintf "wan1"
        set dstintf "lan"
        set srcaddr "MGMT-NET-V6"
        set dstaddr "ALL_IPv6"
        set action accept
        set schedule "always"
        set service "SSH"
        set logtraffic all
        next
end
```

## ICMPv6 Policy

```bash
# Create ICMPv6 service objects
config firewall service custom
    edit "ICMPV6-PTB"
        set protocol ICMP6
        set icmptype 2
        set comment "Packet Too Big — required for PMTUD"
        next
    edit "ICMPV6-ECHO"
        set protocol ICMP6
        set icmptype 128
        set comment "Echo Request"
        next
end

# Allow ICMPv6 Packet Too Big through firewall
config firewall policy6
    edit 50
        set name "Allow-ICMPv6-PTB"
        set srcintf "any"
        set dstintf "any"
        set srcaddr "ALL_IPv6"
        set dstaddr "ALL_IPv6"
        set action accept
        set service "ICMPV6-PTB"
        set comments "Packet Too Big — NEVER block"
        next
end
```

## IPv6 Bogon Filtering

```bash
# Create address objects for bogon prefixes
config firewall address6
    edit "BOGON-DOC"
        set type ipprefix
        set ip6 2001:db8::/32
        next
    edit "BOGON-ULA"
        set type ipprefix
        set ip6 fc00::/7
        next
end
config firewall addrgrp6
    edit "BOGON-V6"
        set member "BOGON-DOC" "BOGON-ULA"
        next
end

# Block bogons at WAN ingress
config firewall policy6
    edit 1
        set name "Block-Bogon-IPv6"
        set srcintf "wan1"
        set dstintf "any"
        set srcaddr "BOGON-V6"
        set dstaddr "ALL_IPv6"
        set action deny
        set logtraffic all
        next
end
```

## Verification

```bash
# Show IPv6 policies
show firewall policy6

# Show active IPv6 sessions
diagnose sys session6 list

# Show IPv6 routing table
get router info6 routing-table

# Debug IPv6 packet flow
diagnose debug flow filter6 addr 2001:db8:ext::1
diagnose debug flow show function-name enable
diagnose debug enable

# Check policy hit counters
show firewall policy6 100
# Shows: packets, bytes, hit count

# Test with ping
execute ping6 2001:db8:server::1
```

## Summary

FortiGate IPv6 policies use the same zone-based framework as IPv4 but configured under `firewall policy6`. Create IPv6 address objects (`firewall address6`) and use them in policies with `srcaddr`/`dstaddr`. Always create an explicit policy allowing ICMPv6 Packet Too Big (type 2) on all paths — FortiGate may block it in a default-deny policy. Enable logging (`logtraffic all`) for security monitoring. Use `diagnose sys session6 list` to see active IPv6 sessions and `diagnose debug flow filter6` to trace specific IPv6 packet flows through the policy engine.
