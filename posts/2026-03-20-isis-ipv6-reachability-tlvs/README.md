# How to Understand IS-IS IPv6 Reachability TLVs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IS-IS, IPv6, TLV, Protocol, Networking

Description: Understand the IS-IS TLVs used to carry IPv6 routing information - specifically TLV 232 (IPv6 Interface Addresses) and TLV 236 (IPv6 Reachability).

## Overview

IS-IS communicates routing information through TLVs (Type-Length-Value structures) embedded in Link State PDUs (LSPs). IPv6 support was added by defining new TLVs, allowing IS-IS to carry IPv6 reachability without changing the core protocol.

## Key IPv6 IS-IS TLVs

| TLV Type | Name | RFC | Purpose |
|----------|------|-----|---------|
| 232 | IPv6 Interface Addresses | RFC 5308 | Carries IPv6 addresses on each interface |
| 236 | IPv6 Reachability | RFC 5308 | Carries reachable IPv6 prefixes |
| 222 | MT-Reachable IS | RFC 5120 | Multi-topology IS neighbor information |
| 235 | MT IPv6 Reachability | RFC 5120 | IPv6 prefixes in multi-topology mode |

## TLV 232: IPv6 Interface Addresses

TLV 232 carries the IPv6 addresses configured on a router's interfaces. This is equivalent to the IP Interface Address TLV (132) for IPv4:

```yaml
TLV 232 Structure:
+------+--------+------------------------------------------+
| Type | Length |  IPv6 Address (16 bytes per address)     |
| 232  |  N*16  |  fe80::1...                              |
+------+--------+------------------------------------------+
```

Both link-local and global addresses are included. The link-local address is used by neighbors to form adjacencies.

## TLV 236: IPv6 Reachability

TLV 236 carries IPv6 prefixes that this router can reach. It is the IPv6 equivalent of TLVs 128/130 (IPv4 Internal/External Reachability):

```yaml
TLV 236 Structure per prefix:
+--------+--------+-------------+-----------+------------------+
| Metric | Up/Down| External Bit| SubTLV len| IPv6 prefix bits |
|  4 bytes|  bit  |    bit      |  1 byte   | (0-128 bits)     |
+--------+--------+-------------+-----------+------------------+
```

The Up/Down bit prevents routing loops between Level 1 and Level 2 areas.

## TLV 235: MT IPv6 Reachability (Multi-Topology)

In multi-topology mode, TLV 235 replaces TLV 236 to include the topology ID:

```yaml
TLV 235 starts with:
+--------+------------------------------------------+
| MT-ID  | followed by same format as TLV 236 entries |
+--------+
```

## Viewing TLVs in IS-IS Database

```text
! Cisco: Show verbose IS-IS database to see TLVs
Router# show isis database verbose

IS-IS Level-2 Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime ATT/P/OL
R1.00-00            * 0x00000017   0xd55b        1196         1/0/0

  IPv6 Interface Address: fe80::1
  IPv6 Interface Address: 2001:db8::1
  MT IPv6 Reachability: 2001:db8:1::/64 Metric: 10
  MT IPv6 Reachability: 2001:db8:2::/64 Metric: 20
```

```bash
# FRRouting: Show IS-IS database with TLV details

vtysh -c "show isis database detail"

# Look for:
# IPv6 Interface Addresses TLV:
#   fe80::1
#   2001:db8::1
# IPv6 Reachability TLV:
#   2001:db8:1::/64 metric 10
```

## Understanding the Up/Down Bit

The U/D (Up/Down) bit in TLV 236 prevents route leaks between levels:
- When a Level-2 prefix is leaked down into a Level-1 area, the U/D bit is set to 1
- If a Level-1/2 router sees this prefix with U/D=1, it does NOT leak it back up to Level-2
- This prevents routing loops in hierarchical IS-IS deployments

## Capturing and Analyzing IS-IS PDUs

```bash
# Capture IS-IS PDUs (Ethertype 0x8870 for IS-IS, or look for protocol 124 in IP header)
sudo tcpdump -i eth0 -n "ether proto 0x8870"

# For IS-IS over point-to-point links:
sudo tcpdump -i eth0 -n "proto isis"

# Open in Wireshark to decode TLVs visually
sudo tcpdump -i eth0 -w /tmp/isis.pcap "ether proto 0x8870"
# In Wireshark: Filter "isis.tlv.code == 236" to show IPv6 reachability TLVs
```

## Summary

IS-IS carries IPv6 routing information in TLV 232 (interface addresses) and TLV 236 (reachable prefixes). In multi-topology mode, TLV 235 replaces TLV 236 to include the topology ID. The Up/Down bit prevents inter-level routing loops. Use `show isis database verbose` to inspect TLV contents and verify IPv6 prefixes are being propagated correctly.
