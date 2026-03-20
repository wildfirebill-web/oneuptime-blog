# How to Configure IPv6 Multicast on Juniper Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, Juniper, PIM-SM, JunOS

Description: A guide to configuring IPv6 multicast routing on Juniper routers running JunOS, including PIM, RP configuration, and MLD settings.

## Enabling IPv6 Multicast Routing on Juniper

```juniper
# Enable PIM protocol for IPv6
set protocols pim family inet6 rp static address 2001:db8::rp

# Configure PIM on interfaces
set protocols pim interface ge-0/0/0.0 mode sparse
set protocols pim interface ge-0/0/1.0 mode sparse
set protocols pim interface lo0.0 mode sparse

# Set PIM version (default is v2)
set protocols pim interface ge-0/0/0.0 version 2

# Configure MLD on subscriber-facing interfaces
set protocols mld interface ge-0/0/0.0 version 2
set protocols mld interface ge-0/0/0.0 query-interval 60
set protocols mld interface ge-0/0/0.0 query-last-member-interval 1
```

## Configuring Static RP

```juniper
# Configure static RP for all multicast groups
set protocols pim family inet6 rp static address 2001:db8::rp

# Configure RP for specific group range
set protocols pim family inet6 rp static address 2001:db8::rp group-ranges ff3e::/32

# Verify RP configuration
run show pim rps inet6
```

## Configuring BSR for Dynamic RP Discovery

```juniper
# Configure Bootstrap Router
set protocols pim family inet6 rp bootstrap priority 200
set protocols pim family inet6 rp bootstrap local address 2001:db8::bsr

# Configure RP candidate
set protocols pim family inet6 rp local address 2001:db8::rp
set protocols pim family inet6 rp local group-ranges ff3e::/32
set protocols pim family inet6 rp local priority 10
```

## Configuring MLD

```juniper
# MLD version and query settings
set protocols mld interface ge-0/0/0.0 version 2
set protocols mld interface ge-0/0/0.0 query-interval 125
set protocols mld interface ge-0/0/0.0 query-last-member-interval 1
set protocols mld interface ge-0/0/0.0 robust-count 2

# Restrict which multicast groups hosts can join
set policy-options prefix-list ALLOWED_MCAST ff3e::/32

set protocols mld interface ge-0/0/0.0 group-limit 100
set protocols mld interface ge-0/0/0.0 group-policy ALLOWED_MCAST
```

## Enabling IPv6 Multicast Scoped Zones

```juniper
# Configure multicast scoped zones to prevent leakage
set protocols pim family inet6 scope-policy SCOPE_POLICY

set policy-options policy-statement SCOPE_POLICY
  term site-local {
    from {
      route-filter ff05::/16 orlonger
    }
    then reject
  }
  term default {
    then accept
  }
```

## Verification Commands

```juniper
# Show PIM neighbors
run show pim neighbors inet6

# Show PIM interface configuration
run show pim interfaces inet6

# Show RP table
run show pim rps inet6

# Show multicast routing table
run show multicast route inet6

# Show detailed multicast route entry
run show multicast route inet6 group ff3e::db8:stream detail

# Show MLD group memberships
run show mld group inet6

# Show MLD statistics
run show mld statistics

# Show PIM join state
run show pim join inet6

# Show PIM statistics
run show pim statistics inet6
```

## Configuring Multicast Traffic Policies

```juniper
# Limit multicast traffic rate (protect against multicast storms)
set class-of-service traffic-control-profiles MCAST_LIMIT
  scheduler-map MCAST_SCHED

set firewall family inet6 filter MCAST_RATE_LIMIT
  term limit-mcast {
    from {
      destination-address ff3e::/32
      protocol 17
    }
    then {
      policer MCAST_POLICER
      accept
    }
  }

set firewall policer MCAST_POLICER
  if-exceeding {
    bandwidth-limit 10m
    burst-size-limit 1500000
  }
  then discard
```

## Debugging IPv6 Multicast on Juniper

```juniper
# Check multicast forwarding cache
run show multicast pim inet6 statistics

# Trace PIM operations
set protocols pim traceoptions file pim-trace.log
set protocols pim traceoptions file size 1m
set protocols pim traceoptions flag packets
set protocols pim traceoptions flag state-transitions

# View trace log
run show log pim-trace.log | last 50

# Clear multicast route cache (use carefully in production)
# run clear multicast route inet6 all

# Check RPF (Reverse Path Forwarding) for a source
run show multicast rpf inet6 2001:db8::source
```

## Juniper vs Cisco IPv6 Multicast Comparison

| Command Function | Cisco IOS | Juniper JunOS |
|---|---|---|
| Enable multicast | `ipv6 multicast-routing` | Configured via protocols |
| Enable PIM on interface | `ipv6 pim` | `set protocols pim interface ge-x/x/x.0` |
| Static RP | `ipv6 pim rp-address` | `set protocols pim family inet6 rp static` |
| Show PIM neighbors | `show ipv6 pim neighbor` | `run show pim neighbors inet6` |
| Show mroute | `show ipv6 mroute` | `run show multicast route inet6` |
| Show MLD groups | `show ipv6 mld groups` | `run show mld group inet6` |

## Summary

Juniper JunOS IPv6 multicast uses the `protocols pim` and `protocols mld` hierarchy. Enable PIM in sparse mode on each interface, configure the RP (static or BSR), enable MLD on subscriber-facing interfaces, and apply access policies to control group membership. Use `show pim neighbors inet6`, `show multicast route inet6`, and `show mld group inet6` to verify the multicast topology.
