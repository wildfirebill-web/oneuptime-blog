# How to Configure IPv6 Firewall Policies on Juniper SRX

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Juniper SRX, Firewall, Security Zones, JunOS

Description: Learn how to configure IPv6 security policies on Juniper SRX firewalls, including zone-based policies, address book entries, stateful inspection, and ICMPv6 handling.

## Overview

Juniper SRX uses zone-based security policies for firewalling. IPv6 traffic is controlled by the same security zones and policies as IPv4 — you simply use IPv6 addresses in the address book and policy rules. SRX performs stateful inspection by default, tracking IPv6 TCP, UDP, and ICMPv6 sessions.

## Security Zone Configuration for IPv6

```
# Define security zones with IPv6 host-inbound services
set security zones security-zone OUTSIDE interfaces ge-0/0/0.0
set security zones security-zone OUTSIDE host-inbound-traffic system-services ping
set security zones security-zone OUTSIDE host-inbound-traffic protocols ospf

set security zones security-zone INSIDE interfaces ge-0/0/1.0
set security zones security-zone INSIDE host-inbound-traffic system-services all
```

## IPv6 Address Book Entries

```
# Add IPv6 addresses to global address book
set security address-book global address INSIDE-NET-V6 2001:db8:lan::/48
set security address-book global address OUTSIDE-ANY-V6 ::/0
set security address-book global address MGMT-V6 fd00:mgmt::/48
set security address-book global address WEB-SERVER-V6 2001:db8:lan::web/128

# Group addresses
set security address-book global address-set TRUSTED-V6 address MGMT-V6
set security address-book global address-set TRUSTED-V6 address INSIDE-NET-V6
```

## Security Policies for IPv6

### Allow IPv6 from Inside to Outside

```
# Allow all outbound IPv6 from inside zone
set security policies from-zone INSIDE to-zone OUTSIDE policy ALLOW-OUTBOUND-V6
    match source-address INSIDE-NET-V6
    match destination-address OUTSIDE-ANY-V6
    match application any
set security policies from-zone INSIDE to-zone OUTSIDE policy ALLOW-OUTBOUND-V6
    then permit
```

### Allow Specific Inbound IPv6

```
# Allow HTTPS inbound from internet to web server
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-HTTPS-V6
    match source-address OUTSIDE-ANY-V6
    match destination-address WEB-SERVER-V6
    match application junos-https
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-HTTPS-V6
    then permit

# Allow SSH from management network only
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-SSH-MGMT-V6
    match source-address MGMT-V6
    match destination-address WEB-SERVER-V6
    match application junos-ssh
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-SSH-MGMT-V6
    then permit
```

### Logging Policy

```
# Enable logging on critical policies
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-HTTPS-V6 then log session-init
set security policies from-zone OUTSIDE to-zone INSIDE policy ALLOW-HTTPS-V6 then log session-close
set security policies from-zone OUTSIDE to-zone INSIDE policy DENY-ALL-V6
    match source-address OUTSIDE-ANY-V6
    match destination-address INSIDE-NET-V6
    match application any
set security policies from-zone OUTSIDE to-zone INSIDE policy DENY-ALL-V6
    then deny
set security policies from-zone OUTSIDE to-zone INSIDE policy DENY-ALL-V6
    then log session-init
```

## ICMPv6 Handling

Juniper SRX automatically handles certain ICMPv6 messages (Neighbor Discovery). For additional control:

```
# Create an application for ICMPv6 Packet Too Big
set applications application ICMPV6-PTB protocol icmp6 icmp-type 2

# Create application for echo
set applications application ICMPV6-ECHO protocol icmp6 icmp-type 128

# Allow ICMPv6 Packet Too Big from any zone
set security policies from-zone any to-zone any policy ICMPV6-PTB-ALLOW
    match source-address any
    match destination-address any
    match application ICMPV6-PTB
set security policies from-zone any to-zone any policy ICMPV6-PTB-ALLOW
    then permit

# Allow ping from monitoring
set security policies from-zone OUTSIDE to-zone INSIDE policy ICMPV6-PING-MONITOR
    match source-address MONITORING-NET
    match destination-address INSIDE-NET-V6
    match application ICMPV6-ECHO
set security policies from-zone OUTSIDE to-zone INSIDE policy ICMPV6-PING-MONITOR
    then permit
```

## Verification

```
# Show security policies
show security policies from-zone OUTSIDE to-zone INSIDE

# Show active flow table (IPv6 sessions)
show security flow session ipv6

# Sample output:
# Session ID: 12345, Policy name: ALLOW-HTTPS-V6/5, State: Stand-alone, Timeout: 1800
# In: 2001:db8:ext::100/54321 --> 2001:db8:srv::web/443;tcp, Conn Tag: 0x0
# Out: 2001:db8:srv::web/443 --> 2001:db8:ext::100/54321;tcp, Conn Tag: 0x0

# Show hit counts per policy
show security policies hit-count from-zone OUTSIDE to-zone INSIDE

# Test packet processing
show security match-policies from-zone OUTSIDE to-zone INSIDE
  source-ip 2001:db8:ext::1 destination-ip 2001:db8:srv::web protocol tcp
  destination-port 443 source-port 54321

# Show zone information
show security zones

# Show IPv6 routing
show route table inet6.0
```

## Summary

Juniper SRX IPv6 firewalling uses security zones and policies with IPv6 addresses in the address book. Create address book entries (`set security address-book global address NAME prefix/len`) and reference them in policies (`match source-address NAME`). SRX performs stateful inspection by default — return traffic for permitted sessions is automatically allowed. Create application objects for ICMPv6 types not covered by default (`set applications application NAME protocol icmp6 icmp-type N`). Verify with `show security flow session ipv6` for active sessions and `show security policies hit-count` for policy statistics.
