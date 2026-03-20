# How to Advertise IPv6 Prefixes via BGP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Prefix Advertisement, Routing, Networking

Description: Learn the correct methods to advertise IPv6 prefixes to BGP neighbors, including aggregate routes, the network statement, and redistribution.

## Overview

Advertising IPv6 prefixes via BGP requires the prefix to either exist in the routing table or be explicitly configured as an aggregate. Advertising the wrong prefixes (too specific or too broad) can cause routing problems or route leaks.

## Method 1: network Statement (Recommended)

The `network` statement advertises a specific prefix if it exists in the routing table:

```bash
# FRRouting — advertise a /48 prefix
vtysh
configure terminal

! First, ensure the prefix exists in the routing table via a null route
ipv6 route 2001:db8:myorg::/48 Null0

! Advertise the prefix via BGP
router bgp 65001
 address-family ipv6 unicast
  network 2001:db8:myorg::/48

 exit-address-family

end
write memory
```

The null route (blackhole) ensures the prefix is always in the routing table, which triggers the `network` statement to advertise it.

## Method 2: Redistribute Connected

Advertise all IPv6 prefixes assigned to interfaces:

```bash
router bgp 65001
 address-family ipv6 unicast
  redistribute connected
 exit-address-family
```

**Caution**: This can advertise too many specific routes. Use with a route map to filter:

```bash
ipv6 prefix-list MY_PREFIXES seq 10 permit 2001:db8:myorg::/48 le 64
route-map CONNECTED_FILTER permit 10
 match ipv6 address prefix-list MY_PREFIXES

router bgp 65001
 address-family ipv6 unicast
  redistribute connected route-map CONNECTED_FILTER
 exit-address-family
```

## Method 3: Aggregate Addresses

Aggregate multiple more-specific routes into a summary:

```bash
router bgp 65001
 address-family ipv6 unicast
  ! Advertise the aggregate /32 only if a more-specific exists in BGP
  aggregate-address 2001:db8::/32 summary-only

 exit-address-family
```

The `summary-only` keyword suppresses the more-specific routes from being advertised, only sending the aggregate.

## Cisco: network Statement

```
Router(config)# router bgp 65001
Router(config-router)# address-family ipv6 unicast

! Advertise a specific prefix
Router(config-router-af)# network 2001:db8:myorg::/48

! Ensure it's in the routing table
Router(config)# ipv6 route 2001:db8:myorg::/48 Null0
```

## Verifying Prefix is Being Advertised

```bash
# FRRouting: check if prefix is in the BGP table and being advertised
vtysh -c "show bgp ipv6 unicast 2001:db8:myorg::/48"
# Look for: ">" (best path) and "*" (valid)

# Show what's being advertised to a specific peer
vtysh -c "show bgp ipv6 unicast neighbors 2001:db8:peer::2 advertised-routes"

# On Cisco:
show bgp ipv6 unicast neighbors 2001:db8:peer::2 advertised-routes
```

## Conditional Advertising

In FRRouting, you can conditionally advertise a prefix based on whether another prefix exists:

```bash
router bgp 65001
 address-family ipv6 unicast
  ! Advertise this aggregate route always
  network 2001:db8:myorg::/48

 exit-address-family
```

## Best Practices for Prefix Advertisement

1. **Advertise aggregates, not specifics** — Announce /32 or /48 aggregates, not /64 host routes
2. **Use null routes** — Always pair network statements with a null route to ensure the prefix is in the RIB
3. **Filter outbound** — Use prefix lists to prevent accidental advertisement of unintended routes
4. **Register your prefix** — Ensure your IPv6 prefix is registered in an IRR database (ARIN, RIPE) and create RPKI ROAs

## Summary

BGP IPv6 prefix advertisement is best done with the `network` statement paired with a null route. Use `aggregate-address summary-only` to suppress more-specific routes. Always combine outbound advertisements with prefix list filtering, and ensure your prefixes are in RPKI ROA records to prevent route hijacking.
