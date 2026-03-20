# How to Troubleshoot IS-IS IPv6 Adjacency Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, Troubleshooting, Adjacency, Networking

Description: Learn how to diagnose and resolve IS-IS adjacency failures in IPv6 environments, covering NET mismatches, authentication issues, and TLV problems.

## Overview

IS-IS adjacency failures prevent IPv6 routes from being exchanged. Common causes include: NET address area mismatch, IS-IS level mismatch, authentication issues, MTU mismatch, and missing `family iso` on interfaces (Juniper).

## Step 1: Check Neighbor State

```
! Cisco
Router# show isis neighbors

System Id      Type Interface   IP Address    State  Holdtime Circuit Id
R2             L2   Gi0/0       10.0.0.2      UP        22     R2.01

! State should be "UP" — "Init" means one-way Hello, "DOWN" means no contact
```

```bash
# FRRouting
vtysh -c "show isis neighbor"

# State values:
# UP = Adjacency formed
# Initial = Received Hellos but not established
# Down = No adjacency
```

## Step 2: Verify IS-IS is Enabled on the Interface

```
! Cisco
Router# show isis interface GigabitEthernet0/0
! Should show: IS-IS is ENABLED for IPv6

! If not enabled:
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 router isis    ! Enable IPv6 IS-IS
```

## Step 3: Check IS-IS Level Compatibility

Both sides must agree on IS-IS level:

```
! Cisco: Check configured level
Router# show isis protocols | include is-type
! Both sides must be Level-1, Level-2, or L1-L2

! Mismatch example:
! Router A: is-type level-2-only
! Router B: is-type level-1-only
! → These will NOT form adjacency (different levels)
```

## Step 4: Verify NET Address (Area Match for L1)

For Level-1 adjacency, both routers must be in the same area:

```
! Router A NET: 49.0001.0000.0000.0001.00 → Area 49.0001
! Router B NET: 49.0002.0000.0000.0002.00 → Area 49.0002
! These cannot form a Level-1 adjacency (different areas)
! For Level-2 only: area difference is acceptable
```

## Step 5: Check Authentication

```
! Cisco: Verify authentication mode on interface
Router# show isis interface GigabitEthernet0/0 | include Auth
! If authentication is configured on one side but not the other → adjacency fails

! Check authentication key matches
Router# show key chain ISIS_KEY | include key-string
```

## Step 6: Check MTU

IS-IS PDUs must fit within the interface MTU:

```
! Cisco: Check IS-IS hello PDU MTU
Router# show isis interface GigabitEthernet0/0 | include MTU
! If LAN hello PDU size > interface MTU → adjacency fails

! Fix: increase MTU or reduce IS-IS hello size
Router(config-if)# isis lsp-mtu 1400   ! Reduce IS-IS PDU size
```

## Step 7: Verify family iso on Juniper

Juniper requires `family iso` on interfaces for IS-IS:

```
# Check if family iso is configured
show interfaces ge-0/0/0.0 | grep iso

# If missing:
set interfaces ge-0/0/0.0 family iso
```

## Step 8: Capture IS-IS Hellos

```bash
# Capture IS-IS PDUs (direct Layer 2, not IP)
sudo tcpdump -i eth0 -n "ether proto 0x8870"

# With verbose IS-IS decode
sudo tshark -i eth0 -Y "isis" -V | grep -A 5 "Hello"

# Look for:
# - Hello PDUs from neighbor
# - Source MAC and System ID in PDU
# - Area address in PDU (must match for L1)
```

## IS-IS Adjacency Troubleshooting Matrix

| Symptom | Cause | Fix |
|---------|-------|-----|
| No IS-IS PDUs | IS-IS not enabled on interface | Add `ipv6 router isis` to interface |
| Init state only | One-way Hello | Check authentication, MTU, area address |
| Level mismatch | Different is-type config | Match is-type on both sides |
| Area mismatch (L1) | Different area in NET | Change NET to same area for L1 peers |
| Auth failure | Key mismatch | Verify key strings are identical |
| family iso missing | Juniper only | Add `family iso` to interface unit |

## Summary

IS-IS IPv6 adjacency failures are usually caused by: level mismatch, area address mismatch for L1 adjacencies, authentication key mismatch, MTU issues, or (on Juniper) missing `family iso` on the interface. Use `show isis neighbor` for state, tcpdump with `ether proto 0x8870` for raw PDU capture, and check authentication and level settings match on both sides.
