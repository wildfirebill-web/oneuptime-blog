# How to Understand MAP-E (Mapping of Address and Port using Encapsulation)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MAP-E, IPv6 Transition, Encapsulation, ISP

Description: An explanation of MAP-E (Mapping of Address and Port using Encapsulation), a stateless IPv6 transition technology that tunnels IPv4 in IPv6 using algorithmic address mapping.

## What Is MAP-E?

MAP-E (Mapping of Address and Port using Encapsulation), defined in RFC 7597, is a stateless IPv6 transition technology for ISPs. Like MAP-T, it uses algorithmic mapping of IPv4 addresses and port ranges into IPv6 prefixes. The difference is that MAP-E uses **encapsulation** (tunneling IPv4 inside IPv6) rather than translation.

## MAP-E vs MAP-T: The Key Difference

Both technologies use the same mapping algorithm. The only difference is the data-plane mechanism:

| Feature | MAP-E | MAP-T |
|---|---|---|
| Transport | IPv4-in-IPv6 encapsulation | IPv6 header translation (SIIT) |
| IP header overhead | +40 bytes (IPv6 header) | 0 extra bytes (header replaced) |
| Border Router function | Decapsulation | Translation |
| MTU impact | Higher (40-byte overhead) | Lower (same size packet) |
| Standard | RFC 7597 | RFC 7599 |

Both share the same mapping rules (RFC 7598) and DHCPv6 options.

## MAP-E Architecture

```mermaid
graph LR
    A[Subscriber Device<br/>IPv4 only] -->|IPv4| B[MAP-E CE<br/>Encapsulate IPv4 in IPv6]
    B -->|IPv6 tunnel to BR| C[IPv6-Only Network]
    C -->|IPv6 softwire| D[MAP-E BR<br/>Decapsulate IPv6 → IPv4]
    D -->|IPv4| E[IPv4 Internet]
```

**CE (Customer Edge)**: The CPE/home router. Encapsulates IPv4 packets in IPv6 using a computed destination address based on MAP rules.

**BR (Border Router)**: The ISP-side router. Decapsulates IPv4 from IPv6 and forwards to the IPv4 internet.

## Stateless Operation

The power of MAP-E (like MAP-T) is that it operates without per-subscriber state at the BR:

- The CE encapsulates each IPv4 packet in IPv6 with a destination that encodes the IPv4 source address and port range
- The BR decapsulates based on the outer IPv6 source address — which algorithmically identifies the subscriber, their IPv4 address, and port range
- No lookup table needed — the information is embedded in the IPv6 address

## MAP-E Mapping Rules

MAP-E rules are distributed to CEs via DHCPv6 (RFC 7598 options). A typical rule set:

```
# Basic Mapping Rule (BMR) - what the CE uses for its own traffic
Rule-IPv6-Prefix: 2001:db8:map::/48
Rule-IPv4-Prefix: 203.0.113.0/24
EA-bits: 16         # 8 bits for IPv4 host, 8 bits for PSID
PSID-offset: 4      # The a value in RFC 7597
BR-address: 2001:db8:br::1

# Default Mapping Rule (DMR) - for traffic to non-MAP destinations
Default-IPv6-Prefix: 64:ff9b::/96  # or BR's /64 prefix
```

## Port Set Allocation Example

With 24-bit IPv4 prefix and 16 EA bits (8 for host, 8 for PSID), each subscriber gets:

```
16 subscribers share each IPv4 address (4-bit PSID)
Port set size per subscriber: (65536 - 1024) / 16 = 4032 ports
(Ports 0-1023 are reserved, so effective port space is 1024-65535)

Example:
IPv4 address: 203.0.113.5
PSID 0: ports 1024-2047
PSID 1: ports 2048-3071
PSID 2: ports 3072-4095
...
PSID 15: ports 15360-16383
```

## Forwarding Between CEs

One advantage of MAP is that CE-to-CE traffic (between subscribers in the same MAP domain) can flow directly without going through the BR:

```
CE-A wants to reach CE-B:
1. CE-A knows CE-B's IPv4 source (from incoming packet)
2. CE-A computes CE-B's IPv6 address using FMR
3. CE-A sends IPv6 packet directly to CE-B
4. No hairpin through the BR needed
```

This is called **hairpin avoidance** and improves performance for peer-to-peer applications.

## Configuring MAP-E on Linux (CE Side)

```bash
# Load ip6_tunnel module for encapsulation
modprobe ip6_tunnel

# Create the MAP-E tunnel interface
# mode ip4ip6: IPv4-in-IPv6 encapsulation
# local: CE's IPv6 address
# remote: BR's IPv6 address
ip tunnel add mape0 mode ip4ip6 \
    local 2001:db8:map:subscriber::1 \
    remote 2001:db8:br::1 \
    encaplimit none

ip link set mape0 up

# Set MTU to account for IPv6 encapsulation overhead (40 bytes)
ip link set mape0 mtu 1460

# Route IPv4 default traffic through the MAP-E tunnel
ip route add default dev mape0

# Configure iptables to restrict outbound ports to allocated PSID range
# Example: PSID 3, port range 3072-4095
iptables -t nat -A POSTROUTING -o eth0 -p tcp \
    -j SNAT --to-source 203.0.113.5:3072-4095
iptables -t nat -A POSTROUTING -o eth0 -p udp \
    -j SNAT --to-source 203.0.113.5:3072-4095
```

## Comparison: MAP-E vs DS-Lite

| Aspect | DS-Lite | MAP-E |
|---|---|---|
| ISP state | Stateful CGN (AFTR) | Stateless (BR) |
| Scalability | Lower (AFTR bottleneck) | Higher (no state at BR) |
| Abuse investigation | Harder (NAT logs required) | Easier (IPv4 deterministic from IPv6) |
| Port flexibility | Full port range | Restricted port set |
| CPE complexity | Lower | Higher |

## Summary

MAP-E combines the stateless scalability of algorithmic address mapping with IPv4-in-IPv6 encapsulation for transport. It eliminates per-subscriber state at the ISP Border Router, making it more scalable than DS-Lite. The trade-off is added MTU overhead (40 bytes for the IPv6 header) and more complex CPE configuration to enforce port set restrictions.
