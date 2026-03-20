# How to Verify BGP Best Path Selection Using show Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, Routing, Best Path, FRR, Cisco, Verification, Troubleshooting

Description: Learn how to use BGP show commands to verify which path is selected as best and understand the BGP best path selection algorithm in action.

---

BGP best path selection determines which route among multiple candidates is installed in the routing table. When traffic is not flowing as expected, verifying path selection is the first troubleshooting step.

## BGP Best Path Selection Order

BGP evaluates these attributes in order; the first decisive comparison wins:

1. **Weight** (Cisco-only; highest wins)
2. **Local Preference** (highest wins; default 100)
3. **Locally originated** (locally originated > learned)
4. **AS Path length** (shortest wins)
5. **Origin** (IGP < EGP < Incomplete)
6. **MED** (lowest wins; compared within the same AS)
7. **eBGP over iBGP** (external > internal)
8. **IGP metric to next-hop** (lowest wins)
9. **Oldest eBGP path** (more stable wins)
10. **Router ID** (lowest wins)

## show Commands for Best Path Verification

### FRR (Linux / VyOS)

```bash
# Show the BGP table for a specific prefix — asterisk (*) = valid, > = best path
vtysh -c "show ip bgp 192.0.2.0/24"

# Example output:
# BGP routing table entry for 192.0.2.0/24
# Paths: (2 available, best #1, table default)
#   Advertised to update-groups:
#     1
#   65100
#     10.0.0.1 from 10.0.0.1 (10.0.0.1)
#       Origin IGP, metric 0, localpref 200, valid, external, best   ← BEST PATH
#   65200 65100
#     10.0.0.2 from 10.0.0.2 (10.0.0.2)
#       Origin IGP, metric 0, localpref 100, valid, external        ← NOT best (lower localpref)

# Show full BGP table
vtysh -c "show ip bgp"

# Show only the best paths (what's in the IP routing table)
vtysh -c "show ip bgp | grep ^>"
```

### Why a Path Was Selected

```bash
# Show detailed path information with the selection reason
vtysh -c "show ip bgp 192.0.2.0/24 bestpath"

# Show path attributes for a specific neighbor
vtysh -c "show bgp neighbor 10.0.0.1 advertised-routes"
vtysh -c "show bgp neighbor 10.0.0.1 received-routes"
```

### Cisco IOS Equivalent

```
! Show BGP table for a prefix
show ip bgp 192.0.2.0/24

! Show the reason for best path selection
show ip bgp 192.0.2.0/24 bestpath-as-path-multipath-relax

! Show all paths and their attributes
show ip bgp 192.0.2.0 255.255.255.0
```

## Manipulating Best Path

```bash
# FRR: increase local preference for paths from a specific neighbor
# (higher localpref = preferred)
vtysh << 'EOF'
conf t
route-map SET-LOCALPREF permit 10
  set local-preference 200
router bgp 65001
  neighbor 10.0.0.1 route-map SET-LOCALPREF in
EOF

# Apply with soft reset (no session disruption)
vtysh -c "clear bgp 10.0.0.1 soft in"

# Verify the new localpref is reflected in the BGP table
vtysh -c "show ip bgp 192.0.2.0/24"
```

## Key Takeaways

- In `show ip bgp`, `*` means valid and `>` marks the best path selected for the routing table.
- Local Preference is the most commonly tuned attribute for iBGP path selection (higher = preferred).
- AS Path length is the most commonly tuned attribute for eBGP path selection (shorter = preferred).
- Use `show ip bgp <prefix>` (not `show ip route`) to see all BGP paths and why one was chosen.
