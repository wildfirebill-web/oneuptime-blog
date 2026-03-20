# How to Understand IPv6 Multicast Forwarding Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, Forwarding, PIM, Network Routing

Description: An explanation of IPv6 multicast forwarding table structure, how entries are created and used by PIM, and how to read and interpret multicast route entries.

## What Is the Multicast Forwarding Table?

The multicast forwarding table (also called the mroute table or multicast routing information base — MRIB) is a separate routing table used to forward multicast packets. Unlike unicast routing, multicast routing is state-based — each active multicast stream creates entries in the forwarding table.

## Entry Types in the Multicast Forwarding Table

Multicast forwarding tables contain two types of entries:

**(*,G) entries** (star-G): Wildcard source entries for any-source multicast groups. Used by PIM-SM for the shared tree (through RP).
```
(*, ff3e::stream) — Forward this group from any source via the RP
```

**(S,G) entries** (source-G): Source-specific entries for known sources. Used after PIM switches to the source tree.
```
(2001:db8::source, ff3e::stream) — Forward this group from this specific source
```

## Reading the Linux Multicast Forwarding Table

```bash
# View the IPv6 multicast forwarding table
ip -6 mroute show

# Example output:
# (2001:db8::source, ff3e::stream)    Iif: eth1    Oifs: eth0
# (*,              ff3e::stream)      Iif: eth0    Oifs: eth1 eth2

# Detailed output
ip -6 mroute show table all

# Check /proc for multicast routing cache
cat /proc/net/ip6_mr_cache
# Format: group source iif oif packets bytes wrong-if
```

## Reading the FRR Multicast Routing Table

```bash
# View multicast routing table in FRR
vtysh -c "show ipv6 mroute"

# Example output:
# Source           Group            Proto  Input      Output     TTL   Uptime
# *                ff3e::stream     PIM    eth1       eth0       0     0:00:30
# 2001:db8::src    ff3e::stream     PIM    eth1       eth0       1     0:00:15

# Show a specific entry
vtysh -c "show ipv6 mroute ff3e::stream"

# Show PIM topology (more detailed)
vtysh -c "show ipv6 pim topology"
```

## Entry State Machine

PIM-SM creates multicast forwarding entries through these states:

```mermaid
flowchart TD
    A[No State] -->|MLD Report at DR| B[(*,G) entry created<br/>Join sent toward RP]
    B -->|Traffic arrives at RP| C[Shared tree active<br/>(*,G) forwarding]
    C -->|DR switches to source tree| D[(S,G) Join sent<br/>toward source]
    D -->|Source tree active| E[(S,G) forwarding<br/>(*,G) pruned]
    E -->|Last receiver leaves| F[Prune sent<br/>Entry removed]
```

## RPF (Reverse Path Forwarding) Check

IPv6 multicast routing uses RPF checks to prevent routing loops. The RPF check verifies that a multicast packet arrives on the interface toward the source:

```bash
# Check RPF for a specific source
vtysh -c "show ipv6 pim rpf 2001:db8::source"
# Expected output:
# RPF information for 2001:db8::source
#   RPF interface: eth1
#   RPF neighbor: 2001:db8::nexthop
#   RPF route: 2001:db8::/64

# If RPF check fails, packets are dropped
# Check routing table for the source
ip -6 route get 2001:db8::source
```

## Understanding (*,G) vs (S,G) in Forwarding Tables

```bash
# In Cisco IOS: show detailed mroute entry
# Cisco output:
# IPv6 Multicast Routing Table
# (*,ff3e::stream), uptime 00:05:00, pim
#   Incoming interface: Tunnel0, RPF neighbor 2001:db8::rp
#   Outgoing interface list:
#     GigabitEthernet0/0, Forward/Sparse 00:05:00
#
# (2001:db8::source,ff3e::stream), uptime 00:02:00, pim
#   Incoming interface: GigabitEthernet0/1, RPF neighbor 2001:db8::upstream
#   Outgoing interface list:
#     GigabitEthernet0/0, Forward/Sparse 00:02:00
```

## Forwarding Table Statistics

```bash
# Check hit counts on multicast routes (Linux)
ip -6 mroute show | grep -E 'Iif|Oif'

# FRR: check packet/byte counts
vtysh -c "show ipv6 mroute count"

# Check kernel multicast cache statistics
cat /proc/net/ip6_mr_cache
# Columns: group source iif oif pkts bytes wrong-if-pkts

# Monitor multicast traffic rate
watch -n 1 'cat /proc/net/ip6_mr_cache'
```

## Clearing the Forwarding Table

```bash
# Clear all multicast routes (forces them to be re-established)
# Use carefully — causes temporary traffic interruption

# FRR
vtysh -c "clear ipv6 mroute"

# Linux kernel cache
# ip6tables -F OUTPUT  (forces cache flush)
# Or restart PIM daemon
systemctl restart frr
```

## Summary

The IPv6 multicast forwarding table contains (*,G) entries for shared trees and (S,G) entries for source trees. Each entry specifies an incoming interface (RPF result) and outgoing interfaces (active receivers). RPF checks prevent loops. Use `ip -6 mroute show` on Linux, `show ipv6 mroute` in FRR/Cisco, and `show multicast route inet6` in Juniper to inspect forwarding tables. Monitor hit counts to verify active forwarding.
