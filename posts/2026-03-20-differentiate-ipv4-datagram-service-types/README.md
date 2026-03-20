# How to Differentiate IPv4 Datagram Service Types

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Quality of Service, ToS, DSCP, TOS Field

Description: The IPv4 Type of Service (ToS) byte encodes service differentiation hints for routers, evolving from precedence bits and delay/throughput/reliability flags to the modern DSCP and ECN fields.

## The ToS Byte Evolution

The 8-bit ToS field at byte 1 of the IPv4 header has been redefined multiple times:

| Standard | Bit Layout | Purpose |
|----------|-----------|---------|
| RFC 791 (original) | 3-bit Precedence + 4 D/T/R/C flags + 1 reserved | Basic service hints |
| RFC 2474 (DSCP) | 6-bit DSCP + 2-bit ECN | Differentiated Services |

## Original ToS Flags (RFC 791)

```text
Bits 3-5 (Precedence): 0=Routine, 7=Network Control
Bit 4: D - Minimize Delay
Bit 5: T - Maximize Throughput
Bit 6: R - Maximize Reliability
Bit 7: C - Minimize Monetary Cost
```

## DSCP and ECN (Modern)

Modern networks use DSCP (Differentiated Services Code Point) in the upper 6 bits:

| DSCP Class | Value | Use Case |
|-----------|-------|----------|
| Default Forwarding (DF) | 0 | Best-effort traffic |
| Expedited Forwarding (EF) | 46 | Voice/video (low latency) |
| AF41 | 34 | Video conferencing |
| CS6 | 48 | Routing protocols |
| CS7 | 56 | Network control |

## Setting DSCP with Python

```python
import socket

# DSCP value 46 (EF) is shifted left 2 bits to occupy bits 7-2 of the ToS byte

DSCP_EF = 46
tos = DSCP_EF << 2  # = 0xB8 = 184

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
# IP_TOS sets the entire ToS/DSCP byte
sock.setsockopt(socket.IPPROTO_IP, socket.IP_TOS, tos)
sock.sendto(b"voice packet", ("192.168.1.10", 5060))
sock.close()

print(f"Sent with DSCP EF (ToS byte=0x{tos:02X})")
```

## Setting DSCP with tc on Linux

```bash
# Mark all traffic from 192.168.1.100 with DSCP AF41 (value 34)
tc qdisc add dev eth0 root handle 1: htb default 10
tc filter add dev eth0 protocol ip parent 1:0 prio 1 \
    u32 match ip src 192.168.1.100/32 \
    action dsfield set 0x88  # AF41 << 2 = 136 = 0x88
```

## Viewing ToS in Captured Traffic

```bash
# tcpdump shows ToS values with -v
tcpdump -i eth0 -v -n ip | grep 'tos'

# Filter for expedited forwarding traffic
tcpdump -i eth0 'ip[1] == 0xb8'
```

## Key Takeaways

- The ToS byte has evolved from RFC 791 precedence/flags to DSCP+ECN (RFC 2474).
- DSCP occupies the top 6 bits; ECN occupies the bottom 2 bits.
- EF (DSCP=46) is used for voice; AF classes for video; CS6/CS7 for routing.
- Set DSCP via `setsockopt(IP_TOS)` in applications or with `tc`/`iptables` on Linux.
