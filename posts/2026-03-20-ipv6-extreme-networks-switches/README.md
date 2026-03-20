# How to Configure IPv6 on Extreme Networks Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Extreme Networks, ExtremeXOS, Switch, Networking, Campus

Description: Configure IPv6 on Extreme Networks switches running ExtremeXOS including VLAN routing, OSPFv3, and Router Advertisements for campus network deployments.

## Introduction

Extreme Networks switches running ExtremeXOS support IPv6 with a configuration syntax that differs from Cisco IOS. This guide covers the key IPv6 commands for ExtremeXOS-based switches (X Series, ExtremeSwitching).

## Step 1: Enable IPv6 Forwarding

```bash
# Enable IPv6 unicast routing
configure forwarding ipv6 enable

# Verify
show forwarding ipv6
```

## Step 2: Configure IPv6 on a VLAN

Unlike Cisco, Extreme uses VLAN names as the primary identifier:

```bash
# Create VLAN and assign IPv6 address
create vlan "employee-vlan" tag 100
configure vlan "employee-vlan" ipaddress 192.168.100.1/24   # IPv4
configure vlan "employee-vlan" ipv6 address 2001:db8:1:100::1/64 add

# Enable IPv6 on the VLAN
enable ipv6 vlan "employee-vlan"

# Verify
show ipv6 vlan "employee-vlan"
```

## Step 3: Configure Router Advertisements

```bash
# Enable RA on the VLAN
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement send enable

# Set RA interval
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement interval 100

# Set router lifetime
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement lifetime 1800

# Configure the prefix to advertise
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement prefix 2001:db8:1:100::/64 \
    valid-lifetime 86400 preferred-lifetime 14400 autonomous on onlink on add

# Set M and O flags (off for SLAAC)
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement managed-flag off
configure ipv6 neighbor-discovery vlan "employee-vlan" router-advertisement other-flag off
```

## Step 4: Configure Static IPv6 Routes

```bash
# Add a static IPv6 route
configure iproute add default 2001:db8:isp::1 vr "VR-Default" ipv6

# Add a specific route
configure iproute add 2001:db8:2::/48 2001:db8:0:1::2 vr "VR-Default" ipv6

# Verify routing table
show iproute ipv6
```

## Step 5: Configure OSPFv3

```bash
# Configure OSPFv3
configure ospfv3 routerid 1.1.1.1

# Add interfaces to OSPFv3 area 0
configure ospfv3 add vlan "employee-vlan" area 0.0.0.0
configure ospfv3 add vlan "uplink-vlan" area 0.0.0.0

# Enable OSPFv3
enable ospfv3

# Verify
show ospfv3 neighbor
show ospfv3 route
```

## Step 6: Configure IPv6 Access Lists

```bash
# Create an IPv6 access list
create access-list ipv6_protect any icmpv6 type any code any permit

# Block specific traffic
create access-list ipv6_deny_ssh 2001:db8:bad::/48 any TCP destPort 22 deny

# Apply to VLAN
configure access-list add ipv6_protect any first vlan "employee-vlan" ingress
```

## Step 7: Configure DHCPv6 Snooping

```bash
# Enable DHCPv6 snooping on a VLAN
configure dhcpv6-snooping vlan "employee-vlan" enable

# Configure trusted ports (uplink to DHCPv6 server)
configure dhcpv6-snooping port 1 trust-port
configure dhcpv6-snooping port 2:1 trust-port
```

## Step 8: Save Configuration

```bash
# Save configuration
save config
```

## Verification Commands

```bash
# Show all IPv6 addresses
show ipv6 interface

# Show IPv6 routing table
show iproute ipv6

# Show IPv6 neighbor table
show ipv6 neighbors

# Show OSPFv3 neighbors
show ospfv3 neighbor

# Show RA configuration
show ipv6 neighbor-discovery router-advertisement vlan "employee-vlan"

# Ping test
ping 2606:4700:4700::1111 vr VR-Default ipv6
```

## Conclusion

ExtremeXOS IPv6 configuration uses VLAN-centric syntax rather than interface-based syntax, reflecting Extreme's network philosophy. Once familiar with the `configure vlan ... ipv6` and `configure ipv6 neighbor-discovery vlan ...` patterns, the remaining features (static routes, OSPFv3, ACLs) follow logically. The `show iproute ipv6` and `show ipv6 neighbors` commands provide the primary verification views for IPv6 connectivity.
