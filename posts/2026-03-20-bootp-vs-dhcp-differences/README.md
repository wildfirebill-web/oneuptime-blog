# How to Understand BOOTP vs DHCP Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, BOOTP, Networking, Protocol History, IP Addressing

Description: BOOTP was the predecessor to DHCP, providing static IP assignment based on MAC address via UDP broadcasts, while DHCP extended it with dynamic leasing, automatic address pools, and a much richer...

## Historical Context

- **BOOTP** (Bootstrap Protocol, RFC 951, 1985): Designed to allow diskless workstations to boot over the network, providing IP address and boot file information.
- **DHCP** (Dynamic Host Configuration Protocol, RFC 2131, 1997): Built on BOOTP's packet format, adding dynamic address pools, lease times, and extensible options.

## Side-by-Side Comparison

| Feature | BOOTP | DHCP |
|---------|-------|------|
| RFC | 951 | 2131 |
| Address assignment | Static (MAC-to-IP table) | Dynamic pools + reservations |
| Lease concept | Permanent | Time-limited, renewable |
| Option support | Limited (8 fixed fields) | Extensive (options 1-255) |
| Configuration update | Manual (server table) | Automatic (pool management) |
| Protocol | UDP 67/68 | UDP 67/68 (same ports) |
| Packet format | Compatible | Extends BOOTP header |

## BOOTP Packet Format (Shared with DHCP)

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  op (1)       |  htype (1)    |  hlen (1)     |  hops (1)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            xid (4)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          secs (2)             |          flags (2)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          ciaddr (4)                           |
|                          yiaddr (4)                           |
|                          siaddr (4)                           |
|                          giaddr (4)                           |
|                          chaddr (16)                          |
|                          sname (64)                           |
|                          file (128)                           |
|                          options (var, 312 in DHCP)           |
```

## How DHCP Extends BOOTP

DHCP uses the same packet format as BOOTP but adds:
1. **Magic cookie** (0x63825363) at the start of the options field, signaling DHCP format.
2. **Option 53** (DHCP message type) to distinguish DHCPDISCOVER, DHCPOFFER, etc.
3. **Option 51** (lease time) - absent in BOOTP (permanent assignments).
4. **Dynamic pools** - server can assign any available IP from a range.

## BOOTP Relay and DHCP

DHCP relay agents (like `ip helper-address` on Cisco) were originally designed for BOOTP and work identically for DHCP because they share the same packet format.

## Is BOOTP Still Used?

BOOTP itself is largely obsolete - modern systems use DHCP. However:
- DHCP servers often support BOOTP requests for legacy compatibility.
- The packet format remains identical, so DHCP and BOOTP traffic is captured by the same `port 67 or port 68` filter.
- ISC dhcpd handles BOOTP requests automatically if `allow bootp` is in the subnet declaration.

## Key Takeaways

- DHCP is a superset of BOOTP sharing the same UDP ports and base packet structure.
- BOOTP used static MAC-to-IP tables; DHCP introduced dynamic address pools with lease times.
- The DHCP magic cookie (0x63825363) distinguishes DHCP from plain BOOTP packets.
- Relay agents handle both BOOTP and DHCP transparently since they use the same format.
