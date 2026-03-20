# How to Verify RIPng Routes on Cisco

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RIPng, Cisco, IPv6, Routing Table, Verification

Description: Learn the Cisco IOS commands to verify RIPng route installation, routing database contents, and interface participation.

## Overview

Verifying RIPng on Cisco requires checking the RIPng process, the routing database, and the IPv6 routing table. Cisco's IOS provides dedicated `show ipv6 rip` commands for this purpose.

## Primary Verification Commands

```text
! Show RIPng process configuration and status
Router# show ipv6 rip

! Show RIPng routing database
Router# show ipv6 rip database

! Show IPv6 routes learned by RIPng
Router# show ipv6 route rip

! Show RIPng on a specific interface
Router# show ipv6 rip RIPNG_PROCESS interface brief
```

## Interpreting show ipv6 rip

```text
Router# show ipv6 rip

RIP process "RIPNG_PROCESS", port 521, multicast-group ff02::9, pid 312
      Administrative distance is 120. Maximum paths is 16
      Updates every 30 seconds, expire after 180, garbage collect after 240 secs
      Split horizon is on; poison reverse is off
      Default routes are not generated
      Periodic updates 1543, trigger updates 12
      Interfaces:
        GigabitEthernet0/0   (passive-interface is not set)
        GigabitEthernet0/1
```

Key fields to check:
- **Updates every 30 seconds** - Normal update interval
- **Split horizon is on** - Loop prevention is active
- **Interfaces** - Confirm all expected interfaces are listed

## Interpreting show ipv6 rip database

```text
Router# show ipv6 rip database

RIP process "RIPNG_PROCESS"
2001:DB8:1::/64 via FE80::1 on GigabitEthernet0/0, metric 1, installed
2001:DB8:2::/64 directly connected, GigabitEthernet0/1
2001:DB8:3::/64 via FE80::2 on GigabitEthernet0/0, metric 2, installed
  expires in 00:02:30
2001:DB8:4::/64 via FE80::2 on GigabitEthernet0/0, metric 3
  expires in 00:01:45
```

- **metric 1** = directly reachable via that neighbor
- **installed** = route was selected for the routing table
- **expires in** = time remaining before route times out (180s total)

## Interpreting show ipv6 route rip

```text
Router# show ipv6 route rip

IPv6 Routing Table - default - 6 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       R - RIPng

R   2001:DB8:3::/64 [120/2]
     via FE80::2, GigabitEthernet0/0
R   2001:DB8:4::/64 [120/3]
     via FE80::2, GigabitEthernet0/0
```

Format: `R <prefix> [AD/metric] via <next-hop>, <interface>`
- **R** = RIPng learned route
- **[120/2]** = Administrative distance 120, metric (hop count) 2

## Debug Commands

```text
! Enable RIPng debugging (caution: verbose)
Router# debug ipv6 rip

! Debug only RIPng events
Router# debug ipv6 rip events

! Debug only RIPng packet contents
Router# debug ipv6 rip database

! Turn off all RIPng debugging
Router# no debug ipv6 rip
```

## Checking for Count-to-Infinity Issues

```text
Router# show ipv6 rip database

! A route appearing with metric 16 indicates it is unreachable
! and is being garbage-collected
2001:DB8:LOST::/64 via FE80::1 on GigabitEthernet0/0, metric 16
  garbage collect
```

## Monitoring Route Updates in Real Time

```text
! Show RIPng counters (sent/received updates)
Router# show ipv6 rip RIPNG_PROCESS interface GigabitEthernet0/0

! Clear RIPng counters to reset statistics
Router# clear ipv6 rip RIPNG_PROCESS

! Force an immediate RIPng update
Router# clear ipv6 route rip
```

## Summary

On Cisco, `show ipv6 rip database` shows all known RIPng routes with metrics and expiry times. `show ipv6 route rip` shows only the routes installed in the IPv6 routing table (best paths). Check metric values - approaching 15 means you're near the hop limit, and 16 means the route is unreachable and being timed out.
