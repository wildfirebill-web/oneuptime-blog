# How to Design IPv4 Supernets for Route Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Supernetting, Route Aggregation, BGP, OSPF, Network Design

Description: Learn how to design IPv4 supernets (covering prefixes) for efficient route aggregation that minimizes routing table size in OSPF and BGP.

## What Is a Supernet?

A supernet is a network prefix that covers a larger address range than a standard subnet — it combines multiple smaller prefixes into one larger covering prefix. Supernetting is the basis of route aggregation.

```
Subnets:        Supernet:
192.168.0.0/24  →  192.168.0.0/22  (covers .0, .1, .2, .3 /24s)
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24
```

## Step 1: Requirements for Clean Supernetting

For a set of networks to aggregate cleanly into a supernet:
1. Networks must be contiguous (no gaps)
2. The block must start on a power-of-2 boundary
3. The total number of networks must be a power of 2

```
VALID aggregation (all conditions met):
  10.1.0.0/24, 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24
  → 10.1.0.0/22  ✓ (contiguous, starts at .0, count=4=2^2)

INVALID aggregation (not power-of-2):
  10.1.0.0/24, 10.1.1.0/24, 10.1.2.0/24
  → Best summary: 10.1.0.0/22 (leaks .3 which you don't own)
  → Or: 10.1.0.0/23 + 10.1.2.0/24 (two prefixes, not one)
```

## Step 2: Design Address Blocks for Clean Aggregation

Start with the supernet and allocate downward:

```python
from ipaddress import ip_network

def design_allocation(supernet_cidr, site_prefix_len, vlan_prefix_len):
    """Divide a supernet into site blocks and VLAN subnets."""
    supernet = ip_network(supernet_cidr)

    print(f"Supernet: {supernet_cidr}")
    print(f"  → {len(list(supernet.subnets(new_prefix=site_prefix_len)))} sites of /{site_prefix_len}")
    print()

    for i, site_block in enumerate(supernet.subnets(new_prefix=site_prefix_len)):
        vlans = list(site_block.subnets(new_prefix=vlan_prefix_len))
        print(f"Site {i+1}: {site_block} ({len(vlans)} VLANs of /{vlan_prefix_len})")
        for j, vlan in enumerate(vlans[:4]):  # Show first 4
            print(f"  VLAN {j+1}: {vlan}")
        if len(vlans) > 4:
            print(f"  ... and {len(vlans)-4} more")

# Design: Americas region (/16) with sites (/21) and VLANs (/24)
design_allocation('10.1.0.0/16', 21, 24)
```

## Step 3: Configure BGP Aggregation

```
! Method 1: Aggregate with summary-only (suppress more-specifics)
router bgp 65001
  network 10.1.0.0 mask 255.255.0.0   ! Announce the supernet
  aggregate-address 10.1.0.0 255.255.0.0 summary-only

! Method 2: Aggregate with a static Null0 route
ip route 10.1.0.0 255.255.0.0 Null0    ! Generate the aggregate route
router bgp 65001
  redistribute static route-map AGGREGATE_ONLY
  !
route-map AGGREGATE_ONLY permit 10
 match ip address prefix-list AGGREGATE
!
ip prefix-list AGGREGATE seq 5 permit 10.1.0.0/16 le 16
```

## Step 4: Configure OSPF Summarization at ABR

```
! At an OSPF Area Border Router (ABR)
! Summarize all area 1 routes (10.1.0.0/21 through 10.1.7.0/21) into one /18

router ospf 1
  ! Summarize to single prefix when entering area 0
  area 1 range 10.1.0.0 255.255.192.0    ! 10.1.0.0/18 summarizes 64x/24s

  ! Only advertise the summary if at least one component route exists
  area 1 range 10.1.0.0 255.255.192.0 not-advertise  ! suppress (use when not needed)
```

## Step 5: Supernet Planning Calculator

```python
from ipaddress import ip_network, collapse_addresses

def supernet_planner(owned_networks, target_prefix_len=None):
    """
    Given a set of owned networks, calculate the supernet and
    determine if any address space is wasted.
    """
    nets = [ip_network(n) for n in owned_networks]
    collapsed = list(collapse_addresses(nets))

    print(f"Owned networks: {owned_networks}")
    print(f"Collapsed: {[str(n) for n in collapsed]}")

    for net in collapsed:
        # Check how many smaller nets are in this collapsed prefix
        subnets_of_common_size = list(net.subnets(new_prefix=max(n.prefixlen for n in nets)))
        owned_in_this = [s for s in subnets_of_common_size if any(s == owned for owned in nets)]
        wasted = len(subnets_of_common_size) - len(owned_in_this)

        if wasted > 0:
            print(f"WARNING: {net} contains {wasted} unowned subnets "
                  f"(advertising this supernet includes space you don't own)")

supernet_planner([
    '10.1.0.0/24', '10.1.1.0/24', '10.1.2.0/24', '10.1.3.0/24'
])
# Clean: 10.1.0.0/22 with 0 waste

supernet_planner([
    '10.1.0.0/24', '10.1.1.0/24', '10.1.3.0/24'
])
# WARNING: 10.1.0.0/22 includes 10.1.2.0/24 which you don't own
```

## Conclusion

IPv4 supernets enable route aggregation by covering multiple smaller prefixes with a single larger prefix. Design address blocks in powers of 2 starting at aligned boundaries to enable clean aggregation. Configure BGP aggregation with `aggregate-address ... summary-only` to suppress more-specifics, and OSPF summarization with `area X range` at ABRs. Always verify with Python's `collapse_addresses()` that your aggregate doesn't accidentally include address space you don't own.
