# How to Understand IPv6 Multicast Address Format

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, Networking, Address Format, RFC 4291

Description: A detailed explanation of the IPv6 multicast address format, including flag bits, scope fields, and the different categories of multicast addresses.

## IPv6 Multicast Address Structure

IPv6 multicast addresses always start with the prefix `ff00::/8`. The high byte `ff` (11111111 in binary) identifies the address as multicast.

The full 128-bit structure is:

```text
|  8 bits  |  4 bits  |  4 bits  |        112 bits          |
|  FF      |  flags   |  scope   |  group ID                |
```

## Flag Bits (bits 8-11)

The 4 flag bits have specific meanings:

```text
Bit 8  (highest): Reserved (must be 0)
Bit 9:  R flag  - Rendezvous Point Embedded (RFC 3956)
Bit 10: P flag  - Prefix-based (RFC 3306)
Bit 11 (lowest): T flag  - Transient (1) or Well-known (0)
```

**T flag = 0**: Well-known multicast address (permanently assigned by IANA)
Examples: `ff02::1` (all nodes), `ff02::2` (all routers)

**T flag = 1**: Transient/dynamic multicast address (not permanently assigned)
Example: `ff3e::8000:1` (transient site-local multicast)

## Scope Field (bits 12-15)

The 4-bit scope field determines the reach of the multicast traffic:

| Scope Value | Name | Description |
|---|---|---|
| 0 | Reserved | Not for use |
| 1 | Interface-local | Loopback only |
| 2 | Link-local | Single network segment |
| 3 | Realm-local | Larger than link-local, implementation-defined |
| 4 | Admin-local | Smallest admin-scoped area |
| 5 | Site-local | Within a site/campus |
| 8 | Organization-local | Entire organization |
| E | Global | Internet-wide multicast |
| F | Reserved | Not for use |

## Common Multicast Address Examples

```text
ff01::1  - All nodes, interface-local scope
ff02::1  - All nodes, link-local scope (most common)
ff02::2  - All routers, link-local scope
ff02::5  - OSPFv3 all routers
ff02::6  - OSPFv3 all DR routers
ff02::9  - RIPng all routers
ff02::a  - EIGRP routers
ff02::d  - PIM routers
ff02::fb - mDNS (multicast DNS)
ff02::1:2 - DHCPv6 relay agents and servers
ff02::1:3 - DHCPv6 all servers
ff05::1:3 - DHCPv6 all servers, site-local scope
ff0e::1  - All nodes, global scope
```

## Constructing Multicast Addresses

To construct a multicast address:

```text
1. Start with: ff
2. Add flags nibble: 0 (well-known) or 1x (transient, set T bit)
3. Add scope nibble: 2 (link-local), 5 (site-local), e (global)
4. Add group ID (112 bits)

Example: Link-local, well-known, group 1
ff = multicast
0  = flags (T=0, well-known)
2  = scope (link-local)
...::1 = group ID 1
Result: ff02::1
```

## Solicited-Node Multicast Address

A special multicast derived from a unicast address, used for Neighbor Discovery:

```text
Format: ff02::1:ff00:0/104 + last 24 bits of unicast address

Example: Unicast 2001:db8::1
Last 24 bits: 00:00:01
Solicited-node: ff02::1:ff00:0001 = ff02::1:ff00:1
```

## Verifying Multicast Group Membership

```bash
# View multicast groups the system has joined

ip -6 maddr show

# Example output:
# 2: eth0
#     inet6 ff02::1           # All nodes
#     inet6 ff02::1:ff00:1    # Solicited-node for 2001:db8::1
#     inet6 ff02::2           # All routers (if routing enabled)

# Check multicast group membership in detail
ip -6 maddr show dev eth0
```

## Prefix-Based Multicast (P flag)

RFC 3306 defines prefix-based multicast addresses where the group ID is derived from a unicast prefix:

```text
Format with P=1, T=1:
ff3<scope>::<plen><prefix><group>

Example: Group for prefix 2001:db8::/32
ff3e:0030:2001:db8::1
(scope=e global, plen=0x20=32, prefix=2001:db8, group=1)
```

## Summary

IPv6 multicast addresses use the `ff00::/8` prefix with flag bits defining whether the address is well-known (T=0) or transient (T=1), and a 4-bit scope field defining traffic reach (link-local=2, site-local=5, global=e). The 112-bit group ID identifies the specific multicast group. Understanding this format is essential for configuring multicast routing, MLD, and network services that use multicast (DHCPv6, NDP, OSPFv3).
