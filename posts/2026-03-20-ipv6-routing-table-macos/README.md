# How to View the IPv6 Routing Table on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, macOS, Routing Table, netstat, Networking

Description: Learn how to view and interpret the IPv6 routing table on macOS using netstat, route, and networksetup commands.

## Overview

macOS uses the BSD-derived `netstat` and `route` commands to inspect the IPv6 routing table. Understanding these outputs helps you diagnose connectivity issues and verify that IPv6 routing is correctly configured.

## Using netstat

```bash
# Show IPv6 routing table (no DNS resolution, verbose flags)
netstat -rn -f inet6

# Sample output:
# Routing tables
#
# Internet6:
# Destination                Gateway          Flags     Netif  Expire
# default                    fe80::1%en0      UGcg       en0
# ::1                        ::1              UHL        lo0
# 2001:db8::/64              link#6           UC          en0
# 2001:db8::1                link#6           UHL        en0
# fe80::%en0/64              link#6           UC          en0
# fe80::aabb:ccdd:eeff%en0   link#6           UHL        en0
# ff01::%en0/32              link#6           UmCI        en0
# ff02::%en0/32              link#6           UmCI        en0
```

## Understanding macOS Route Flags

| Flag | Meaning |
|------|---------|
| U | Route is up |
| G | Route uses a gateway (next hop) |
| H | Host route (single address, not a network) |
| C | Route was cloned from a parent route |
| c | Cloning enabled |
| m | Multicast route |
| I | Interface route |
| L | Link-level (ARP/NDP) information |

## Using the route Command

```bash
# Show details about a specific route
route -n get -inet6 2001:db8::100

# Output:
#    route to: 2001:db8::100
# destination: 2001:db8::
#      gateway: link#6
#    interface: en0
#        flags: <UP,DONE,CLONING,STATIC>
#   recvpipe  sendpipe  ssthresh  rtt,msec    rttvar  hopcount      mtu     expire
#         0         0         0         0         0         0      1500       0

# Show the default IPv6 route
route -n get -inet6 default
```

## Using networksetup

```bash
# Show IPv6 configuration for an interface
networksetup -getinfo "Wi-Fi"
networksetup -getinfo "Ethernet"

# Output includes:
# IPv6: Automatic
# IPv6 IP address: 2001:db8::1
# IPv6 Router: fe80::1
```

## Using ifconfig for Interface-Specific IPv6

```bash
# Show all IPv6 addresses assigned to en0
ifconfig en0 inet6

# Show only global scope addresses
ifconfig en0 inet6 | grep "scopeid 0x0"

# Full interface details
ifconfig -a inet6
```

## System Information Tool

```bash
# Get network info including IPv6 routing via system_profiler
system_profiler SPNetworkDataType | grep -A 10 "IPv6"
```

## Verifying the Default Gateway

```bash
# Confirm the IPv6 default gateway
netstat -rn -f inet6 | grep "^default"

# The entry should show your router's link-local address
# default    fe80::1%en0    UGcg    en0
```

## Live Route Monitoring

macOS doesn't have a built-in live route monitor like Linux's `ip monitor`, but you can watch changes with:

```bash
# Poll routing table changes every 2 seconds
watch -n 2 "netstat -rn -f inet6"

# Or use route monitor (shows kernel routing messages)
route -n monitor
```

## Summary

On macOS, `netstat -rn -f inet6` is the primary command for viewing the IPv6 routing table. Use `route -n get -inet6 <address>` to simulate a route lookup for a specific destination, and `networksetup -getinfo <interface>` to check the configured IPv6 gateway from a higher-level perspective.
