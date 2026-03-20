# How to Understand BGP Path Selection for IPv4 Prefixes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, Networking, Routing, IPv4, Path Selection, FRR

Description: Understand BGP's multi-step path selection algorithm and how attributes like weight, local preference, AS path length, and MED determine which route is chosen.

## Introduction

When BGP receives multiple paths to the same IPv4 prefix from different peers, it applies a deterministic selection algorithm to choose the single best path to install in the routing table. Understanding this process lets you engineer traffic flows precisely using BGP attributes.

## The BGP Path Selection Algorithm

BGP evaluates attributes in strict order — the first differentiating attribute wins:

```
1. Weight (Cisco-proprietary, highest wins — local to router only)
2. Local Preference (highest wins — propagated within AS)
3. Locally originated (prefer routes originated by this router)
4. AS Path length (shortest wins)
5. Origin code (IGP < EGP < Incomplete)
6. MED / Multi-Exit Discriminator (lowest wins — sent to eBGP peers)
7. eBGP over iBGP (prefer externally learned)
8. IGP metric to next-hop (lowest wins)
9. Oldest eBGP route (prefer more stable)
10. Router ID (lowest BGP Router ID wins — tiebreaker)
11. Neighbor IP address (lowest — final tiebreaker)
```

## Viewing BGP Path Selection in FRR

```bash
# Show all paths for a prefix and which is selected (marked with >)
vtysh -c "show ip bgp 10.10.0.0/24"

# Example output:
# BGP routing table entry for 10.10.0.0/24
# Paths: (2 available, best #1)
#   Advertised to non peer-group peers: 10.0.0.2
#   65100
#     10.0.0.1 from 10.0.0.1 (10.0.0.1)
#       Origin IGP, metric 0, localpref 200, weight 0, valid, external, best
#   65200 65300
#     10.0.0.2 from 10.0.0.2 (10.0.0.2)
#       Origin IGP, metric 0, localpref 100, weight 0, valid, external
```

## Influencing Path Selection

### Local Preference (prefer outbound path)

```bash
# Set higher local preference for routes from preferred upstream
route-map PREFER-ISP1 permit 10
  set local-preference 200

route-map PREFER-ISP2 permit 10
  set local-preference 100

router bgp 65001
  neighbor 203.0.113.1 route-map PREFER-ISP1 in  # ISP1 preferred
  neighbor 198.51.100.1 route-map PREFER-ISP2 in
```

### AS Path Prepending (influence inbound traffic)

```bash
# Prepend own AS to make this path less attractive to remote peers
route-map DEPREF-OUTBOUND permit 10
  set as-path prepend 65001 65001 65001

router bgp 65001
  neighbor 198.51.100.1 route-map DEPREF-OUTBOUND out
```

### MED (influence which entry point remote AS uses)

```bash
# Advertise lower MED out the preferred link
route-map SET-MED-LOW permit 10
  set metric 100

route-map SET-MED-HIGH permit 10
  set metric 500

router bgp 65001
  neighbor 203.0.113.1 route-map SET-MED-LOW out
  neighbor 198.51.100.1 route-map SET-MED-HIGH out
```

## Verifying the Selected Path

```bash
# Show why a specific path was selected
vtysh -c "show ip bgp 10.10.0.0/24 bestpath"

# Show BGP summary — number of prefixes and neighbor status
vtysh -c "show ip bgp summary"

# Show the installed route in the kernel RIB
ip route show 10.10.0.0/24
```

## Conclusion

BGP path selection is deliberate and attribute-driven. Local preference controls outbound preferences within your AS, AS path prepending signals inbound preferences to remote networks, and MED fine-tunes which entry points remote peers use. Knowing the selection order lets you predict which path BGP will choose and configure attributes to achieve your traffic engineering goals.
