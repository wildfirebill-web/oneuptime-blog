# How to Understand MAP-T (Mapping of Address and Port using Translation)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MAP-T, IPv6 Transition, ISP, Address Mapping

Description: A clear explanation of MAP-T (Mapping of Address and Port using Translation), a stateless IPv6 transition technology that encodes IPv4 addresses and port ranges into IPv6 addresses algorithmically.

## What Is MAP-T?

MAP-T (Mapping of Address and Port using Translation) is defined in RFC 7599. It is a stateless IPv6 transition technology designed for ISP deployments that:

- Provides IPv4 connectivity over an IPv6-only access network
- Is **stateless** — no NAT state tables needed at the ISP level
- Uses **algorithmic mapping** between IPv4 addresses and port ranges and IPv6 prefixes
- Uses **translation** (not encapsulation) to convert between IPv4 and IPv6

MAP-T is similar to DS-Lite in goals but eliminates the need for per-subscriber state in the ISP network, significantly improving scalability.

## MAP-T vs MAP-E vs DS-Lite

| Technology | Transport | Stateful? | ISP device |
|---|---|---|---|
| DS-Lite | IPv4-in-IPv6 tunnel | Stateful CGN (AFTR) | AFTR |
| MAP-E | IPv4-in-IPv6 encapsulation | Stateless | BR (Border Router) |
| MAP-T | IPv6 header translation | Stateless | BR (Border Router) |

## The Stateless Mapping Concept

The key innovation in MAP-T is that both the CPE and the ISP Border Router compute IPv4↔IPv6 mappings **from a rule** — no state table is needed. Given a subscriber's IPv6 prefix, you can deterministically compute:

- Which IPv4 address the subscriber uses
- Which port range they are assigned

This is the "Mapping of Address and Port" — ports are divided among subscribers sharing the same IPv4 address.

## MAP-T Rule Parameters

A MAP-T deployment uses "MAP rules" pushed to CPEs via DHCPv6. Each rule specifies:

- **Rule IPv6 prefix (Rule IPv6 Prefix + EA bits)**: The subscriber's IPv6 space
- **Rule IPv4 prefix**: The shared IPv4 address space
- **EA bits (Embedded Address bits)**: How many bits encode the IPv4 address and port set index
- **PSID offset (a)**: Port Set Identifier offset
- **BR address**: IPv6 address of the ISP Border Router

## Port Set Division

With MAP-T, multiple subscribers share a single public IPv4 address. The public port range is divided:

```
IPv4 address: 203.0.113.1 (shared by 16 subscribers)
Port range (subscriber 0): 1024-2047
Port range (subscriber 1): 2048-3071
Port range (subscriber 2): 3072-4095
...
```

The PSID (Port Set Identifier) identifies which subscriber "owns" a given port range. A 4-bit PSID means 16 subscribers share each IPv4 address.

## Address Synthesis in MAP-T

For a subscriber with IPv6 prefix `2001:db8:0000::/56` and MAP rule:

```
Rule IPv6 Prefix: 2001:db8::/40
Rule IPv4 Prefix: 203.0.113.0/24
EA bits: 16 (8 for IPv4 address + 8 for PSID)
```

The CE (CPE) IPv6 address encodes:
- The ISP's /40 prefix
- The subscriber's IPv4 address bits
- The PSID (port set identifier)

This allows the BR to compute the IPv4 destination from the IPv6 source address without any lookup table.

## Packet Translation Flow

```mermaid
sequenceDiagram
    participant APP as App on Device
    participant CE as MAP-T CE (CPE)
    participant NET as IPv6 Network
    participant BR as MAP-T BR (ISP)
    participant IPV4 as IPv4 Internet

    APP->>CE: IPv4 packet (src: 203.0.113.5:1024, dst: 8.8.8.8:80)
    CE->>CE: Translate IPv4 to IPv6 (SIIT + MAP address synthesis)
    CE->>NET: IPv6 packet (src: 2001:db8:..., dst: 64:ff9b::808:808)
    NET->>BR: IPv6 packet forwarded to BR
    BR->>BR: Translate IPv6 to IPv4 (reverse SIIT)
    BR->>IPV4: IPv4 packet (src: 203.0.113.5:1024, dst: 8.8.8.8:80)
```

## MAP-T Domain vs Default Mapping Rule

A MAP-T domain consists of:
- **BMR (Basic Mapping Rule)**: Maps the subscriber's IPv6 prefix to IPv4 addresses/ports
- **FMR (Forwarding Mapping Rule)**: Rules for forwarding between CE devices in the same domain
- **DMR (Default Mapping Rule)**: For traffic to destinations outside the MAP domain (uses BR address)

The DMR uses the NAT64 well-known prefix or a configured prefix to translate external IPv4 destinations to IPv6 addresses that route to the BR.

## Advantages of MAP-T

- **Stateless**: No session tables at the BR — each packet translates independently
- **Scalable**: BR handles millions of subscribers without state
- **Auditable**: IPv4 addresses and ports map deterministically to IPv6 addresses, simplifying abuse investigation
- **No tunneling overhead**: Translation adds no extra headers (unlike DS-Lite or MAP-E)

## Limitations

- **Port restrictions**: Subscribers share IPv4 addresses with limited port ranges
- **Protocols without ports**: GRE, ICMP need special handling
- **Complexity**: MAP rules and EA bit calculation are non-trivial to configure
- **Limited vendor support**: Fewer CPE devices support MAP-T than DS-Lite

## Summary

MAP-T provides stateless IPv4-over-IPv6 connectivity for ISPs by algorithmically mapping IPv4 addresses and port ranges into IPv6 addresses. Unlike DS-Lite, it requires no per-subscriber state at the ISP's Border Router, making it highly scalable. The trade-off is configuration complexity and port range restrictions for subscribers sharing a single IPv4 address.
