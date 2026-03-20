# How to Understand Recursive Routing Lookups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Routing, BGP, Recursive Lookup, IPv4, Troubleshooting

Description: Understand recursive routing lookups - where a route's next-hop requires another routing lookup - and how misconfigurations cause routes to become invalid.

## Introduction

A recursive routing lookup occurs when a route's next-hop is not directly connected and must itself be resolved through another routing lookup. BGP routes commonly require recursive lookups: the BGP next-hop might be a loopback address resolved via an IGP route. Understanding this chain is essential for diagnosing why routes become invalid.

## How Recursive Lookups Work

```text
BGP route: 10.20.0.0/24 next-hop 192.0.2.1
           |
           +--> Recursive lookup for 192.0.2.1
                |
                +--> OSPF route: 192.0.2.0/30 via 172.16.0.1 dev eth0
                     |
                     +--> Connected: 172.16.0.0/24 dev eth0  (resolved!)
```

The final resolved next-hop is 172.16.0.1 on eth0. If the OSPF route for 192.0.2.0/30 disappears, the BGP route for 10.20.0.0/24 becomes invalid.

## Viewing Recursive Resolution in FRR

```bash
# Show a BGP route with its recursive next-hop resolution

vtysh -c "show ip bgp 10.20.0.0/24"

# Look for "nexthop... resolved via..." in the output
# Example:
#   10.20.0.0/24 unicast
#     192.0.2.1 (inaccessible)  <-- recursive lookup failed!
#     OR
#     192.0.2.1 (metric 10) (accessible via 172.16.0.1)
```

## Diagnosing a Failed Recursive Lookup

```bash
# Check if the BGP next-hop is resolvable
ip route get 192.0.2.1

# If no route: "RTNETLINK answers: Network is unreachable"
# That's why the BGP route is invalid

# Check what IGP route covers the next-hop
ip route show | grep "192.0.2"

# Check OSPF for the covering route
vtysh -c "show ip ospf route" | grep "192.0.2"
```

## Maximum Recursion Depth

Linux and FRR limit recursive lookups to prevent infinite chains. If your next-hop chain is too deep, routes may not resolve:

```bash
# FRR allows up to 10 recursive lookups by default
# Check BGP next-hop recursion status
vtysh -c "show ip bgp nexthop detail"
```

## BGP Next-Hop Resolution via Connected Route Only

Some configurations require BGP next-hops to be resolvable only via connected routes (no IGP recursion):

```bash
# FRR BGP: disable recursive next-hop resolution
router bgp 65001
  no bgp recursive-ebgp-best-path

# Or restrict next-hop to connected check
neighbor 10.0.0.2 ebgp-multihop 1
```

## Static Route Recursive Lookup

```bash
# Static route with recursive next-hop
# 10.20.0.0/24 via 192.168.5.1
# 192.168.5.1 must be reachable via a connected or other route

ip route add 10.20.0.0/24 via 192.168.5.1

# If 192.168.5.1 is not in the routing table, the route is rejected:
# Error: Nexthop has invalid gateway.
```

## Conclusion

Recursive routing lookups form dependency chains between routes. A failure at any level in the chain invalidates all routes that depend on it. This is why IGP stability is critical in networks running BGP - if OSPF loses a route to a BGP next-hop, all BGP prefixes pointing to that next-hop become unreachable. Monitor your IGP health to protect BGP reachability.
