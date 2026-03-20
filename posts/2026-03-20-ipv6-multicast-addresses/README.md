# How to Understand IPv6 Multicast Addresses (ff00::/8)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Networking, Multicast, Addressing, NDP

Description: Understand the structure, scopes, and common uses of IPv6 multicast addresses in the ff00::/8 range including well-known multicast groups.

## Introduction

IPv6 multicast addresses (ff00::/8) replace both IPv4 multicast and broadcast. In IPv6, there is no broadcast - every use case that relied on broadcast in IPv4 now uses multicast. Understanding multicast is essential for IPv6 networking, as protocols like NDP, DHCPv6, and MLD all rely on it.

## Multicast Address Structure

An IPv6 multicast address starts with `ff` and has this layout:

```text
| 8 bits | 4 bits | 4 bits | 112 bits  |
|  0xff  | flags  | scope  | Group ID  |
```

- **Flags (4 bits)**: `0` = well-known IANA address, `1` = transient/dynamically assigned
- **Scope (4 bits)**: Defines the reach of the multicast group
- **Group ID**: Identifies the multicast group

## Scope Values

The scope nibble controls how far multicast traffic can propagate:

| Scope | Value | Description |
|---|---|---|
| Interface-local | 1 | Loopback only, never forwarded |
| Link-local | 2 | Single link, not forwarded by routers |
| Realm-local | 3 | Larger than link but topologically determined |
| Admin-local | 4 | Smallest administratively scoped region |
| Site-local | 5 | Single site, not beyond site boundary |
| Organization-local | 8 | Across an entire organization |
| Global | e | No boundary (internet-wide) |

## Well-Known Multicast Addresses

```text
ff01::1  - All nodes (interface-local)
ff01::2  - All routers (interface-local)
ff02::1  - All nodes (link-local)        ← most common
ff02::2  - All routers (link-local)      ← used by RS messages
ff02::5  - All OSPFv3 routers
ff02::6  - OSPFv3 DR/BDR routers
ff02::9  - All RIPng routers
ff02::a  - All EIGRP routers
ff02::d  - All PIM routers
ff02::16 - All MLDv2-capable routers
ff02::1:2 - All DHCPv6 servers/relay agents
ff02::1:3 - All DHCPv6 servers
ff05::1:3 - All DHCPv6 servers (site-local)
```

## Solicited-Node Multicast Addresses

A special type of link-local multicast used by NDP for address resolution (replacing ARP). Every unicast or anycast address has a corresponding solicited-node multicast address:

```text
Solicited-node prefix: ff02::1:ff00:0/104

# The last 24 bits of the unicast address are appended

# Example: 2001:db8::1234:5678
# Solicited-node: ff02::1:ff34:5678
```

```python
# Python: calculate solicited-node multicast address
import ipaddress

def solicited_node(unicast_addr):
    addr = ipaddress.IPv6Address(unicast_addr)
    # Take the last 24 bits (6 hex chars)
    addr_int = int(addr)
    last_24 = addr_int & 0xFFFFFF
    # Combine with solicited-node prefix ff02::1:ff00:0
    prefix = 0xff020000000000000000000100ff0000
    sn_addr = ipaddress.IPv6Address(prefix | last_24)
    return sn_addr

print(solicited_node("2001:db8::1234:5678"))
# Output: ff02::1:ff34:5678
```

## Viewing Multicast Group Membership on Linux

```bash
# Show all multicast group memberships on all interfaces
ip -6 maddr show

# Show for a specific interface
ip -6 maddr show dev eth0

# Using netstat (older method)
netstat -g -n
```

Example output:
```text
2:      eth0
        inet6 ff02::1
        inet6 ff02::1:ff00:1
        inet6 ff01::1
```

## Multicast Listener Discovery (MLD)

MLD is the IPv6 equivalent of IGMP for managing multicast group memberships. Routers use MLD to track which hosts on a link want which multicast groups:

```bash
# Check MLD statistics on Linux
cat /proc/net/igmp6

# Monitor MLD messages with tcpdump
sudo tcpdump -i eth0 -vv "ip6[6] == 0 and ip6[40] == 58 and (ip6[42] == 130 or ip6[42] == 131 or ip6[42] == 143)"
```

## Conclusion

IPv6 multicast addresses are fundamental to how IPv6 networks function. Unlike IPv4 where multicast was optional, IPv6 depends on multicast for neighbor discovery, router advertisements, and DHCPv6. Understanding scope values and well-known addresses helps with troubleshooting, firewall configuration, and designing IPv6 network infrastructure.
