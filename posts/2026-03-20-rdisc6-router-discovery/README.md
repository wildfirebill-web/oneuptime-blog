# How to Use rdisc6 for Router Discovery Diagnostics - Router Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Rdisc6, Router Discovery, NDP, Router Advertisement, Network Diagnostics

Description: Use rdisc6 to send Router Solicitation messages and analyze Router Advertisement responses for IPv6 default gateway and prefix configuration diagnostics.

## Introduction

`rdisc6` sends IPv6 Router Solicitation messages and displays the Router Advertisement (RA) responses from routers on the local link. It reveals IPv6 prefixes being advertised, router preferences, MTU settings, and flags that control how hosts configure their IPv6 addresses. This is essential for diagnosing SLAAC and DHCPv6 configuration issues.

## Installation

```bash
# Ubuntu/Debian (part of ndisc6 package)

sudo apt install -y ndisc6

# RHEL/CentOS
sudo yum install -y ndisc6
```

## Basic Usage

```bash
# Send Router Solicitation and display received RA
rdisc6 eth0

# Multiple attempts (useful if first RA is missed)
rdisc6 -m 3 eth0

# Wait longer for RA (milliseconds)
rdisc6 -w 5000 eth0

# Use a specific source IPv6 address
rdisc6 eth0 fe80::10

# Verbose output
rdisc6 -v eth0
```

## Understanding Router Advertisement Output

```text
Soliciting ff02::2 (All-routers on eth0)...

Hop limit                 :   64 (      0x40)
Stateful address conf.    :   No
Other stateful conf.      :   No
Mobile home agent         :   No
Router preference         :   medium
Neighbor discovery proxy  :   No
Router lifetime           : 1800 (0x00000708) seconds
Reachable time            :    0 (0x00000000) milliseconds
Retransmit time           :    0 (0x00000000) milliseconds
 Prefix                   : 2001:db8:cafe::/64
  Valid time              : 86400 seconds
  Pref. time              : 14400 seconds
  On-link                 :  Yes
  Autonomous address conf.:  Yes
 Source link-layer address: 52:54:00:ab:cd:ef
 from fe80::5054:ff:feab:cdef
```

Key fields:
- **Hop limit**: Default TTL/hop limit for this router's network
- **Stateful address conf.**: `M` flag - if Yes, use DHCPv6 for addresses
- **Other stateful conf.**: `O` flag - if Yes, use DHCPv6 for other config (DNS, NTP)
- **Router lifetime**: How long this router is valid as default gateway (0=not a router)
- **Prefix**: IPv6 prefix for SLAAC address generation
- **Autonomous address conf.**: `A` flag - if Yes, generate address via SLAAC

## Diagnosing SLAAC Issues

```bash
# Check what prefixes are being advertised
rdisc6 eth0 2>/dev/null | grep "Prefix\|Autonomous\|Stateful"

# Expected for SLAAC:
# Prefix: 2001:db8::/64
# Autonomous address conf.: Yes

# If Autonomous = No, SLAAC won't generate addresses
# If no prefix is advertised, hosts can't generate global addresses
```

## Diagnosing DHCPv6 Triggers

```bash
# Check M and O flags in Router Advertisement
rdisc6 eth0 2>/dev/null | grep "Stateful"

# M flag = 1 means use DHCPv6 for addresses
# O flag = 1 means use DHCPv6 for DNS/other options
# Output:
# Stateful address conf.    :   Yes  → DHCPv6 needed for addresses
# Other stateful conf.      :   Yes  → DHCPv6 needed for DNS
```

## Capturing and Analyzing All RAs

```bash
# Monitor for all Router Advertisements on the network
sudo tcpdump -n -i eth0 icmp6 and ip6[40] == 134 -v

# Decode the RA fields (134 = RA type)
sudo tcpdump -n -i eth0 icmp6 and ip6[40] == 134 -vvv 2>/dev/null

# With rdisc6, wait and capture multiple RAs
rdisc6 -m 5 -w 10000 eth0 2>/dev/null
```

## Diagnosing "No Default Route" Issues

```bash
#!/bin/bash
# diagnose-ipv6-gateway.sh

echo "=== IPv6 Router Discovery Diagnostics ==="

# 1. Current default route
echo "Current default route:"
ip -6 route show default

# 2. Discover routers via rdisc6
echo ""
echo "Discovering IPv6 routers..."
RA_OUTPUT=$(rdisc6 -m 2 -w 3000 eth0 2>/dev/null)

if [ -z "$RA_OUTPUT" ]; then
    echo "ERROR: No Router Advertisement received!"
    echo "  - Check if a router is present on the link"
    echo "  - Check if RA guard/filtering is blocking RAs"
    echo "  - Try: sudo ip6tables -L FORWARD -n (check for RA blocking)"
else
    echo "Router Advertisement received!"
    echo "$RA_OUTPUT" | grep -E "Prefix|Lifetime|Stateful|from"
fi

# 3. Check current RA acceptance settings
echo ""
echo "RA acceptance (accept_ra):"
for iface in $(ls /proc/sys/net/ipv6/conf/ | grep -v "^all$\|^default$\|^lo$"); do
    val=$(cat /proc/sys/net/ipv6/conf/$iface/accept_ra 2>/dev/null)
    echo "  $iface: $val (0=ignore, 1=accept, 2=accept+forwarding)"
done
```

## Checking RA Guard in Network

```bash
# Some switches implement RA Guard which blocks RAs from unauthorized ports
# Test by sending RA from a port that should be guarded:

# Check if RA is blocked by capturing on a different host
# (This requires physical access or a packet capture tool)

# On the same host, check if RAs are being filtered by ip6tables
sudo ip6tables -L FORWARD -n | grep "ICMPv6\|icmpv6"
sudo ip6tables -L INPUT -n | grep "ICMPv6\|icmpv6"

# Check for rogue RA prevention (radvd or NetworkManager config)
grep -r "IgnoreIf\|interface" /etc/radvd.conf 2>/dev/null
```

## Verifying RA-Assigned Addresses Were Created

```bash
# After running rdisc6, check if SLAAC address was created
PREFIX=$(rdisc6 eth0 2>/dev/null | grep "Prefix" | awk '{print $3}' | head -1)
echo "Prefix from RA: $PREFIX"

# Check if address was auto-configured from the prefix
ip -6 addr show eth0 | grep "${PREFIX%::*}"
```

## Conclusion

`rdisc6` is the diagnostic tool for understanding what IPv6 configuration your routers are advertising. Use it to verify that SLAAC prefixes are being distributed, check whether DHCPv6 is required (M/O flags), and diagnose why hosts aren't getting IPv6 default routes (Router lifetime = 0 or no RA received). When hosts have no IPv6 default route, `rdisc6` quickly reveals whether the problem is a misconfigured router or a blocked Router Advertisement.
