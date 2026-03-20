# How to Configure OSPFv3 Stub Areas for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPFv3, IPv6, Stub Areas, OSPF, Routing

Description: Learn how to configure OSPFv3 stub and totally stub areas to reduce LSA flooding and simplify routing in branch office environments.

## Overview

OSPFv3 stub areas reduce routing overhead in branch locations that only need a default route to reach the rest of the network. In a stub area, AS-external LSAs (Type 5) are not flooded, and the ABR injects a default route instead.

## Stub Area Types

| Area Type | Type 5 LSAs | Type 3 LSAs | Default Route |
|-----------|-------------|-------------|---------------|
| Regular | Yes | Yes | Optional |
| Stub | No | Yes | Yes (auto-injected) |
| Totally Stub | No | No (except default) | Yes |

## Configuring a Stub Area on Cisco IOS

```
! Configure Area 1 as a stub area on ALL routers in that area
! This must be configured identically on all routers in the area

! On the ABR (connecting Area 0 to Area 1)
Router-ABR(config)# router ospfv3 1
Router-ABR(config-router)# address-family ipv6 unicast
Router-ABR(config-router-af)# area 1 stub
! ABR automatically injects default route (Type 3 LSA) into Area 1

! On the internal router in Area 1
Router-Branch(config)# router ospfv3 1
Router-Branch(config-router)# address-family ipv6 unicast
Router-Branch(config-router-af)# area 1 stub
```

## Configuring a Totally Stub Area on Cisco

A totally stub area blocks both external (Type 5) and inter-area (Type 3) LSAs. Only one Type 3 LSA (the default route) is allowed in:

```
! On the ABR ONLY — add no-summary to create totally stub
Router-ABR(config)# router ospfv3 1
Router-ABR(config-router)# address-family ipv6 unicast
Router-ABR(config-router-af)# area 1 stub no-summary

! Internal router still uses just 'stub' (not no-summary)
Router-Branch(config)# router ospfv3 1
Router-Branch(config-router)# address-family ipv6 unicast
Router-Branch(config-router-af)# area 1 stub
```

## Configuring Stub Areas on FRRouting

```bash
vtysh
configure terminal

! Configure Area 1 as stub on all routers in the area
router ospf6
 area 0.0.0.1 stub

! Configure as totally stub (no-summary) — ABR only
router ospf6
 area 0.0.0.1 stub no-summary

end
write memory
```

## Customizing the Default Route Cost

The ABR injects a default route into the stub area. You can control its cost:

```
! Cisco: Set default route cost for stub area
Router-ABR(config)# router ospfv3 1
Router-ABR(config-router)# address-family ipv6 unicast
Router-ABR(config-router-af)# area 1 default-cost 100
```

```bash
# FRRouting: Set default-metric for the stub area
vtysh
configure terminal
router ospf6
 area 0.0.0.1 default-cost 100
end
```

## Verifying Stub Area Configuration

```
! Cisco: Verify stub area flag in OSPF database
Router# show ospfv3 database summary

! Verify the default route is in the branch router's routing table
Router-Branch# show ipv6 route ospf
! Should show: OI ::/0 [110/100] via FE80::ABR, <interface>
```

```bash
# FRRouting: Verify stub area configuration
vtysh -c "show ipv6 ospf"
# Should show: Area 0.0.0.1 is a stub area

# Verify default route in branch routing table
ip -6 route show default proto ospf
```

## Benefits of Stub Areas

- **Reduced LSDB size** — External LSAs are not flooded into the area
- **Reduced memory usage** — Fewer LSAs stored on branch routers
- **Simplified routing** — Branch routers only need a default route, not full topology
- **Faster convergence** — Smaller LSDB means faster SPF calculations

## Summary

OSPFv3 stub areas prevent AS-external LSAs from flooding into branch areas. Configure `area <id> stub` on all routers in the area — the ABR automatically injects a default route. Use `stub no-summary` (totally stub) on the ABR to also block inter-area LSAs, giving branch routers only a single default route.
