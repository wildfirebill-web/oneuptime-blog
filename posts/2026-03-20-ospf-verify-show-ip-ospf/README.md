# How to Verify OSPF Operation with show ip ospf Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPF, Verification, Routing, FRR, Cisco, IPv4, Troubleshooting, Show Commands

Description: Learn how to use OSPF show commands to verify neighbor adjacencies, LSDB contents, route propagation, and overall OSPF health on FRR and Cisco routers.

---

Systematic use of `show ip ospf` commands is the primary method for verifying OSPF configuration and operation. This guide covers the most important commands and what to look for in their output.

## 1. Overall OSPF Status

```bash
# FRR

vtysh -c "show ip ospf"

# Key things to check:
# - Router ID: is it what you expect?
# - Number of areas
# - SPF schedule
# - External LSA count

# Cisco IOS
show ip ospf
```

## 2. OSPF Neighbor Adjacencies

```bash
# FRR: List all OSPF neighbors and their state
vtysh -c "show ip ospf neighbor"

# Output columns:
# Neighbor ID | Pri | State | Dead Time | Address | Interface
# 10.0.0.2    | 1   | Full/ DR | 00:00:35 | 192.168.1.2 | eth0

# States to look for:
# Full = healthy adjacency
# 2-Way = neighbor seen but not adjacent (normal on multi-access networks for non-DR/BDR)
# Init/ExStart/Exchange = adjacency forming
# Down = no recent hellos received

# Full details on a specific neighbor
vtysh -c "show ip ospf neighbor 10.0.0.2 detail"
```

## 3. OSPF Interface Status

```bash
# FRR: Check OSPF is running on the right interfaces with correct parameters
vtysh -c "show ip ospf interface"

# Verify:
# - Correct IP address and subnet mask (mask mismatch = no adjacency)
# - Correct area assignment
# - Hello/Dead intervals (both ends must match)
# - DR/BDR election result

vtysh -c "show ip ospf interface eth0"
```

## 4. Link-State Database (LSDB)

```bash
# FRR: Show the full LSDB
vtysh -c "show ip ospf database"

# Show only specific LSA types
vtysh -c "show ip ospf database router"      # Type 1: Router LSAs
vtysh -c "show ip ospf database network"     # Type 2: Network LSAs
vtysh -c "show ip ospf database summary"     # Type 3: Summary LSAs
vtysh -c "show ip ospf database external"    # Type 5: External LSAs
vtysh -c "show ip ospf database nssa-external" # Type 7: NSSA LSAs

# Verify:
# - All routers in an area should have identical Type 1 and 2 LSAs
# - Type 3 LSAs propagate between areas via ABRs
```

## 5. OSPF Routes in the Routing Table

```bash
# FRR: Show OSPF-derived routes
vtysh -c "show ip route ospf"
# or
vtysh -c "show ip route | grep ^O"

# Route type codes:
# O      = OSPF intra-area
# O IA   = OSPF inter-area (came through ABR)
# O E1   = OSPF external type 1
# O E2   = OSPF external type 2
```

## 6. SPF Calculation Statistics

```bash
# Check how often SPF is running (frequent SPF = instability)
vtysh -c "show ip ospf"
# Look for: SPF algorithm last executed X.XXX seconds ago
# and: SPF minimum holdtime X ms
```

## Cisco IOS Quick Reference

```text
show ip ospf                     ! Overall OSPF status
show ip ospf neighbor            ! Neighbor adjacencies
show ip ospf interface brief     ! Interface participation summary
show ip ospf database            ! Full LSDB
show ip route ospf               ! OSPF routes in routing table
show ip ospf statistics          ! SPF and LSA statistics
```

## Key Takeaways

- Always start verification with `show ip ospf neighbor` - `Full` state is the goal for every expected adjacency.
- `show ip ospf interface` reveals configuration parameters like hello intervals and area assignments that must match between neighbors.
- All routers in an area must have identical LSDBs; compare `show ip ospf database` output across routers if you suspect database inconsistency.
- `O IA` routes come from another area; `O E1/E2` routes come from redistribution.
