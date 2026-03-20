# How to Configure BGP Dampening to Suppress Flapping Routes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, Dampening, Route Flapping, Cisco IOS, Routing Stability

Description: Learn how to configure BGP route dampening to suppress unstable routes that repeatedly appear and disappear, protecting your network from routing instability.

## What Is BGP Route Flapping?

A BGP route flap occurs when a prefix is withdrawn and re-advertised repeatedly—typically due to an unstable link or interface. Each flap causes UPDATE messages to propagate across the Internet, consuming CPU and bandwidth. BGP dampening penalizes flapping routes by suppressing them temporarily.

## How Dampening Works

Each time a route flaps (is withdrawn), its penalty increases by 1000 (default). The penalty decays exponentially over time:
- **Suppress limit (2000):** Route is suppressed when penalty exceeds this value
- **Reuse limit (750):** Route is re-advertised when penalty drops below this value
- **Half-life (15 min):** Time for penalty to drop to half its current value
- **Max suppress time (60 min):** Maximum time a route stays suppressed

## Step 1: Enable BGP Dampening

Enable dampening globally for all BGP routes using default values:

```
router bgp 65001
 ! Enable dampening with default parameters
 bgp dampening
```

Or configure custom parameters:

```
router bgp 65001
 ! Custom: half-life=15min, reuse=750, suppress=2000, max-suppress=60min
 bgp dampening 15 750 2000 60
```

## Step 2: Apply Dampening Selectively with Route Maps

Apply dampening only to specific prefixes using a route map:

```
! Define which prefixes to dampen
ip prefix-list DAMPEN_THESE seq 10 permit 0.0.0.0/0 ge 25

! Route map applies dampening for long prefixes (/25 and longer)
route-map DAMPEN_LONG_PREFIXES permit 10
 match ip address prefix-list DAMPEN_THESE
 ! Use non-default parameters for these prefixes
 set dampening 5 750 2000 20

route-map DAMPEN_LONG_PREFIXES permit 20
 ! Everything else gets no dampening

router bgp 65001
 bgp dampening route-map DAMPEN_LONG_PREFIXES
```

## Step 3: View Dampened Routes

```
! Show all currently dampened routes
Router# show ip bgp dampened-paths

Status codes: d damped, h history, * valid, > best, i internal

   Network          From             Reuse    Path
d  192.168.99.0/24  203.0.113.1      00:45:00  65100 65200 i

! The 'Reuse' column shows how long until the route is re-advertised
```

```
! Show history - routes that had penalties but aren't suppressed yet
Router# show ip bgp flap-statistics

   Network          From        Flaps Duration  Reuse  Path
*  192.168.50.0/24  10.0.0.1        5  00:10:00  -     65100 i
```

## Step 4: Check a Route's Current Penalty

```
Router# show ip bgp 192.168.99.0/24

BGP routing table entry for 192.168.99.0/24
  Dampinfo: penalty 2400, flapped 3 times in 00:05:00
           reuse in 00:45:00
           Current penalty 2400 exceeds suppress limit 2000
```

## Step 5: Manually Clear Dampening for a Route

If a route has been stabilized and you want to re-advertise it immediately:

```
! Clear dampening for a specific prefix
Router# clear ip bgp dampening 192.168.99.0/24

! Clear all dampened routes
Router# clear ip bgp dampening
```

## Dampening Considerations

- **Over-dampening:** Overly aggressive settings can suppress legitimate routes for too long, especially during maintenance
- **RIPE Recommendation:** RIPE-229 recommends conservative dampening settings to avoid instability during routing protocol reconvergence
- **Default is off:** Dampening is not enabled by default because improper tuning can cause more harm than good
- **Monitor regularly:** Check `show ip bgp flap-statistics` to identify genuinely unstable neighbors

## Conclusion

BGP dampening protects your network from route flap instability by penalizing repeatedly withdrawn prefixes. Enable it with conservative default parameters, use route maps for selective application, and monitor dampened paths with `show ip bgp dampened-paths`. Always have a process to manually clear dampening when a genuinely stable route gets incorrectly suppressed.
