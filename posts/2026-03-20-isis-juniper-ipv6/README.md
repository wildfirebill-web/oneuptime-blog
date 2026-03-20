# How to Configure IS-IS on Juniper for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, Juniper, IPv6, JunOS, Routing

Description: Step-by-step guide to configuring IS-IS for IPv6 routing on Juniper routers using JunOS, including multi-topology and per-interface metric configuration.

## Overview

Juniper JunOS natively supports IS-IS for both IPv4 and IPv6. IPv6 reachability is enabled by activating the `ipv6-unicast` topology. The configuration is clean and follows JunOS's hierarchical structure.

## Basic IS-IS IPv6 Configuration

```
# Set the NET (ISO network address)
set routing-options router-id 1.1.1.1
set protocols isis interface lo0.0 passive

# Enable IPv6 topology in IS-IS
set protocols isis topologies ipv6-unicast

# Enable IS-IS on interfaces
set protocols isis interface ge-0/0/0.0 level 2 metric 10
set protocols isis interface ge-0/0/1.0 level 2 metric 10
set protocols isis interface lo0.0 passive

# Set IS-IS level
set protocols isis level 1 disable    ! Level-2 only
```

## Full IS-IS IPv6 Configuration

```
protocols {
    isis {
        level 1 disable;              # Level-2 only operation
        interface ge-0/0/0.0 {
            level 2 {
                metric 10;
            }
        }
        interface ge-0/0/1.0 {
            level 2 {
                metric 10;
            }
        }
        interface lo0.0 {
            passive;                   # No hellos on loopback
        }
        topologies ipv6-unicast;       # Enable IPv6 topology
    }
}

interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                address 10.0.0.1/30;
            }
            family inet6 {
                address 2001:db8:1::1/64;
            }
            family iso;               # Required for IS-IS
        }
    }
    lo0 {
        unit 0 {
            family inet { address 1.1.1.1/32; }
            family inet6 { address 2001:db8::1/128; }
            family iso { address 49.0001.0001.0001.0001.00; }
        }
    }
}
```

## Setting IPv6-Specific Interface Metrics

```
# Set different metrics for IPv6 topology (per-interface)
set protocols isis interface ge-0/0/0.0 topologies ipv6-unicast metric 20
set protocols isis interface ge-0/0/1.0 topologies ipv6-unicast metric 15
```

## IS-IS Authentication

```
# Authentication per-interface
set protocols isis interface ge-0/0/0.0 level 2 authentication-key "secretkey"
set protocols isis interface ge-0/0/0.0 level 2 authentication-type md5

# Or area-wide authentication
set protocols isis level 2 authentication-key "areakey"
set protocols isis level 2 authentication-type md5
```

## Verification Commands

```
# Show IS-IS adjacencies
show isis adjacency

# Show IS-IS adjacency detail
show isis adjacency detail

# Show IS-IS link-state database
show isis database

# Show IS-IS IPv6 routes
show route protocol isis table inet6.0

# Show IS-IS statistics
show isis statistics

# Show IS-IS interface status
show isis interface
```

## Sample Output

```
user@router> show isis adjacency

Interface       System         L State     Hold (secs) SNPA
ge-0/0/0.0      R2             2  Up             23  0:0:5e:0:1:2

user@router> show route protocol isis table inet6.0

inet6.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:2::/64        *[IS-IS/18] 00:45:12, metric 20
                        > to 2001:db8:1::2 via ge-0/0/0.0
2001:db8:3::/48        *[IS-IS/18] 00:30:05, metric 30
                        > to 2001:db8:1::2 via ge-0/0/0.0
```

AD 18 = IS-IS Level 2 for IPv6 (Juniper uses 18 for L2 IS-IS, 15 for L1)

## Summary

Juniper IS-IS for IPv6 requires `family iso` on interfaces, `topologies ipv6-unicast` in the IS-IS protocol block, and IPv6 addresses on interfaces. Set per-interface IPv6 metrics with `topologies ipv6-unicast metric`. Verify with `show isis adjacency` and `show route protocol isis table inet6.0`.
