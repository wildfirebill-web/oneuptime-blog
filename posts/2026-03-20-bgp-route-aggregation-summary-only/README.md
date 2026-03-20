# How to Configure BGP Route Aggregation with Summary-Only

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, Route Aggregation, Cisco IOS, Routing, Summarization

Description: Learn how to configure BGP route aggregation to advertise summary prefixes instead of individual component routes, reducing routing table size with the summary-only option.

## Why Aggregate BGP Routes?

Advertising individual /24 prefixes from a /16 allocation contributes to global routing table bloat. Aggregating them into a single summary prefix reduces the number of routes your AS advertises, makes your routing policy cleaner, and is good Internet citizenship.

## How BGP Aggregation Works

The `aggregate-address` command creates a summary route in the BGP table. By default, both the summary and the component routes are advertised. The `summary-only` keyword suppresses the individual more-specific routes.

## Step 1: Ensure Component Routes Are in the BGP Table

Aggregation only works when at least one more-specific route exists in the BGP table. Verify this first:

```text
Router# show ip bgp | include 192.168

! You should see individual routes like:
! *> 192.168.1.0/24   0.0.0.0         32768 i
! *> 192.168.2.0/24   0.0.0.0         32768 i
! *> 192.168.3.0/24   0.0.0.0         32768 i
```

## Step 2: Configure the Aggregate Address

Create an aggregate that covers the range you want to summarize:

```text
router bgp 65001
 ! Aggregate all /24s within 192.168.0.0/16
 aggregate-address 192.168.0.0 255.255.0.0

 ! The above advertises BOTH the aggregate AND the components
 ! Add summary-only to suppress component routes:
 aggregate-address 192.168.0.0 255.255.0.0 summary-only
```

With `summary-only`, only `192.168.0.0/16` is advertised to neighbors; the individual /24s are suppressed.

## Step 3: Use as-set to Preserve AS-Path Information

By default, the aggregate route has an empty AS path (locally originated). Use `as-set` to include a set of all AS numbers from the component routes-this preserves routing policy information:

```text
router bgp 65001
 ! Include AS-path information from all component routes
 aggregate-address 192.168.0.0 255.255.0.0 summary-only as-set
```

The aggregate will now carry an AS_SET attribute listing all contributing ASes, preventing certain routing loops.

## Step 4: Selectively Suppress Component Routes

Instead of suppressing all components, use an unsuppress map to continue advertising specific prefixes alongside the summary:

```text
! Define which prefixes to still advertise despite summary-only
ip prefix-list KEEP_ADVERTISED seq 10 permit 192.168.1.0/24

route-map UNSUPPRESS permit 10
 match ip address prefix-list KEEP_ADVERTISED

router bgp 65001
 aggregate-address 192.168.0.0 255.255.0.0 summary-only
 ! Re-advertise 192.168.1.0/24 to this specific neighbor
 neighbor 203.0.113.1 unsuppress-map UNSUPPRESS
```

## Step 5: Verify the Aggregate in the BGP Table

```text
Router# show ip bgp 192.168.0.0/16

BGP routing table entry for 192.168.0.0/16
  Paths: (1 available)
    Local, (aggregated by 65001 1.1.1.1)
      0.0.0.0 from 0.0.0.0 (1.1.1.1)
        Origin IGP, localpref 100, weight 32768
        Atomic aggregate          <- present when summary-only used
```

The `Atomic aggregate` flag indicates component routes have been suppressed.

## Step 6: Verify Suppressed Routes

```text
! Check that component routes are suppressed (shown with 's' flag)
Router# show ip bgp

Status codes: s suppressed, d damped, h history, * valid, > best
!  s 192.168.1.0/24  0.0.0.0  32768 i   <- suppressed
!  s 192.168.2.0/24  0.0.0.0  32768 i   <- suppressed
! *> 192.168.0.0/16  0.0.0.0  32768 i   <- aggregate advertised
```

## Conclusion

BGP route aggregation with `summary-only` reduces the number of prefixes advertised by your AS. Always ensure component routes exist in the BGP table before creating an aggregate, use `as-set` to preserve routing policy, and verify suppression with the `s` flag in `show ip bgp`.
