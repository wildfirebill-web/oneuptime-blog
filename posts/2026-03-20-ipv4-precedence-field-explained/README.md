# How to Understand the IPv4 Precedence Field

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, QoS, Precedence, ToS, DSCP

Description: The 3-bit Precedence field in the original IPv4 ToS byte allowed traffic prioritization from Routine to Network Control, forming the conceptual foundation for modern DSCP-based QoS.

## What Is the Precedence Field?

In the original RFC 791 definition of IPv4, the Type of Service byte was structured as follows:

```text
Bits 0-2: Precedence
Bit  3:   D (Minimize Delay)
Bit  4:   T (Maximize Throughput)
Bit  5:   R (Maximize Reliability)
Bit  6:   C (Minimize Cost)
Bit  7:   Reserved
```

The 3-bit Precedence field allowed 8 priority levels:

| Value | Name | Typical Use |
|-------|------|-------------|
| 0 | Routine | Normal traffic |
| 1 | Priority | Network management |
| 2 | Immediate | Voice/signaling |
| 3 | Flash | Video |
| 4 | Flash Override | Military priority |
| 5 | Critical | Routing protocols |
| 6 | Internetwork Control | Network infrastructure |
| 7 | Network Control | Highest priority |

## Legacy vs Modern (DSCP)

RFC 2474 (1998) superseded RFC 791's ToS field with Differentiated Services (DiffServ). The 6-bit DSCP field occupies bits 0–5 of the same byte, with bit positions 0–2 corresponding to the former Precedence bits (making DSCP backward-compatible with Precedence for Class Selector codepoints).

## Class Selector DSCP Values (Backward Compatibility)

| Class Selector | DSCP Value | Precedence Equivalent |
|---------------|-----------|----------------------|
| CS0 | 0 | 0 (Routine) |
| CS1 | 8 | 1 (Priority) |
| CS2 | 16 | 2 (Immediate) |
| CS3 | 24 | 3 (Flash) |
| CS4 | 32 | 4 (Flash Override) |
| CS5 | 40 | 5 (Critical) |
| CS6 | 48 | 6 (Internetwork Control) |
| CS7 | 56 | 7 (Network Control) |

## Setting Precedence/DSCP in Python

```python
import socket

# Set DSCP CS5 (Critical, formerly Precedence 5)

DSCP_CS5 = 40          # 0b101000
tos_byte = DSCP_CS5 << 2  # Shift into top 6 bits = 0xA0

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_TOS, tos_byte)
sock.sendto(b"critical control message", ("10.0.0.1", 514))
sock.close()
print(f"Sent with DSCP CS5, ToS=0x{tos_byte:02X}")
```

## Marking with iptables

```bash
# Mark routing protocol traffic (OSPF, protocol 89) with CS6
iptables -t mangle -A OUTPUT -p 89 -j DSCP --set-dscp-class CS6
```

## Key Takeaways

- The original 3-bit Precedence field had 8 priority levels (Routine to Network Control).
- DSCP Class Selectors (CS0–CS7) are backward-compatible with the original Precedence values.
- Most modern networks configure QoS using DSCP, not the original Precedence bits directly.
- Routing protocol packets should be marked CS6/CS7 to ensure they are not dropped during congestion.
