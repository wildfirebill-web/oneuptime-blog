# How to Configure OSPF Areas and Area Types

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPF, Networking, Routing, FRR, Areas, IPv4

Description: Configure OSPF areas including backbone, stub, totally stubby, and NSSA area types to optimize routing table size and limit LSA flooding in large networks.

## Introduction

OSPF divides networks into areas to limit the scope of Link State Advertisement (LSA) flooding and reduce routing table size. Every OSPF network must have a backbone area (Area 0). Other areas connect to Area 0 via Area Border Routers (ABRs) and can be configured as different types to restrict what routing information they receive.

## OSPF Area Types

| Area Type | External Routes | Summary LSAs | Use Case |
|---|---|---|---|
| Backbone (Area 0) | Yes | Yes | Core transit area |
| Standard | Yes | Yes | General purpose |
| Stub | No | Yes | Branch sites with one ABR |
| Totally Stubby | No | No (default only) | Very simple branches |
| NSSA | Redistributed only | Yes | Sites with external routes but no full OSPF external |

## Configuring Area 0 (Backbone)

```bash
# /etc/frr/frr.conf - backbone router
router ospf
  router-id 1.1.1.1
  network 192.168.0.0/24 area 0
  network 10.0.0.0/30 area 0
  # All interfaces in area 0 participate in the backbone
```

## Configuring a Standard Area

```bash
# ABR connecting area 0 and area 1
router ospf
  router-id 2.2.2.2
  network 10.0.0.0/30 area 0      # link to backbone
  network 172.16.0.0/24 area 1    # link to area 1
```

## Configuring a Stub Area

A stub area does not accept Type 5 external LSAs. The ABR injects a default route instead:

```bash
# On the ABR (area 1 is stub)
router ospf
  area 1 stub

# On all routers inside area 1 (all must agree)
router ospf
  area 1 stub
```

## Configuring a Totally Stubby Area

Even more restrictive — blocks both Type 3 summary LSAs and Type 5 external LSAs:

```bash
# On the ABR only (internal routers use regular stub config)
router ospf
  area 1 stub no-summary
  # Internal routers only receive a default route from the ABR
```

## Configuring an NSSA

NSSA allows a non-backbone area to import external routes as Type 7 LSAs, which the ABR translates to Type 5 for the backbone:

```bash
# On the NSSA ABR
router ospf
  area 2 nssa

# On all routers in area 2
router ospf
  area 2 nssa

# Redistribute connected routes into NSSA
router ospf
  redistribute connected
```

## Verifying OSPF Areas

```bash
# Show OSPF area information
vtysh -c "show ip ospf"

# Show LSA database for a specific area
vtysh -c "show ip ospf database"

# Show area-specific summary routes
vtysh -c "show ip ospf database summary"

# Show OSPF neighbor states (should be FULL for adjacencies)
vtysh -c "show ip ospf neighbor"

# Show OSPF routes installed
vtysh -c "show ip ospf route"
```

## Conclusion

OSPF areas are the primary mechanism for scaling OSPF networks. Stub and totally stubby areas dramatically reduce routing table size on branch routers by replacing all external or summary routes with a single default route. Choose area types based on whether routers in that area need to reach specific external destinations or can use a default route for everything outside the area.
