# How to Configure Firewall Rules for IPv4 on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: pfSense, Firewall, IPv4, Rules, Security, pf, FreeBSD

Description: Configure pfSense firewall rules for IPv4 traffic using the GUI, covering LAN-to-WAN, WAN inbound filtering, inter-VLAN policies, and floating rules with logging.

## Introduction

pfSense firewall rules use the FreeBSD pf engine. Rules are evaluated top-to-bottom on each interface; the first match wins. By default, pfSense blocks all inbound WAN traffic and allows all outbound LAN traffic.

## Default Rule Behavior

```
WAN:  Block all inbound (implicit deny)
LAN:  Allow all outbound (default allow rule)
OPT:  Block all (no default allow — must add rules)
```

## Allow Specific Inbound WAN Traffic

Navigate to **Firewall > Rules > WAN > Add**:

```
Action:          Pass
Interface:       WAN
Address Family:  IPv4
Protocol:        TCP
Source:          any
Destination:     WAN address
Destination port: 443 (HTTPS)
Description:     Allow HTTPS inbound
```

## LAN Rules — Restrict Outbound

Navigate to **Firewall > Rules > LAN**:

```
# Rule 1: Allow established (automatic via stateful)
# Rule 2: Allow LAN to internet (HTTP/HTTPS only for restricted users)
Action:     Pass
Source:     LAN net
Destination: any
Port:       80, 443
Protocol:   TCP

# Rule 3: Block everything else from LAN
Action:     Block
Source:     LAN net
Destination: any
Log:        checked
```

## Inter-VLAN Firewall Rules

```
# On VLAN10 (corporate) interface — allow to servers, block to guest
Rule 1: Pass TCP from VLAN10 net to VLAN20 net port 443 (HTTPS only)
Rule 2: Block IP from VLAN10 net to VLAN30 net (guest isolation)
Rule 3: Pass IP from VLAN10 net to any (internet access)

# On VLAN30 (guest) interface — internet only
Rule 1: Block IP from VLAN30 net to RFC1918 (192.168.0.0/16, 10.0.0.0/8)
Rule 2: Pass IP from VLAN30 net to any (internet)
```

## Block RFC1918 on WAN (Anti-Spoofing)

Navigate to **Firewall > Rules > WAN > Add** (top of list):

```
Action:     Block
Source:     10.0.0.0/8
Destination: any
Description: Block RFC1918 from WAN (anti-spoof)

[Repeat for 172.16.0.0/12 and 192.168.0.0/16]
```

## Aliases for Cleaner Rules

Navigate to **Firewall > Aliases > Add**:
```
Name:    WEB_SERVERS
Type:    Host(s)
Network: 192.168.1.100, 192.168.1.101, 192.168.1.102
```

Then use the alias in rules instead of individual IPs.

## pfSense pf Syntax (Diagnostics > Command Prompt)

```bash
# View generated rules
pfctl -sr

# View NAT rules
pfctl -sn

# Show state table
pfctl -s state

# View rule statistics
pfctl -vs rules

# Flush states (use carefully)
pfctl -F states
```

## Conclusion

pfSense firewall rules process top-to-bottom; place specific rules before general ones. Always add a logging block rule at the bottom to capture unmatched traffic. Use aliases to manage groups of IPs cleanly. Block RFC1918 inbound on WAN as anti-spoofing protection, and add explicit allow rules on any new interface since pfSense denies all traffic by default.
