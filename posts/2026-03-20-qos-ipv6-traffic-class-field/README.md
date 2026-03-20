# How to Understand QoS with IPv6 Traffic Class Field

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, QoS, Traffic Class, DSCP, Quality of Service, Networking

Description: Understand the IPv6 Traffic Class field and how it maps to DSCP values for Quality of Service marking, enabling differentiated service treatment across IPv6 networks.

---

The IPv6 Traffic Class field is the QoS marking mechanism for IPv6, equivalent to the IPv4 ToS/DSCP field. Understanding its structure enables correct QoS implementation for voice, video, and data prioritization on IPv6 networks.

## IPv6 Header Traffic Class Field

```yaml
IPv6 Header Structure (40 bytes fixed):
+-------+--------+-------------------+
| Ver(4)| TC(8)  | Flow Label (20)   |  <- First 32 bits
+-------+--------+-------------------+
...

Traffic Class (TC) Field: 8 bits
Bits 0-5: DSCP (Differentiated Services Code Point)
Bits 6-7: ECN (Explicit Congestion Notification)

Same 8-bit layout as IPv4 DSCP/ECN field in DS field
```

## DSCP Values in IPv6 Traffic Class

```text
DSCP Codepoints (6 bits) and their meanings:

Class Selector (CS):
CS0  = 000000 = 0x00 = 0  (Default/Best Effort)
CS1  = 001000 = 0x08 = 8  (Scavenger/Background)
CS2  = 010000 = 0x10 = 16 (OAM)
CS3  = 011000 = 0x18 = 24 (Broadcast Video)
CS4  = 100000 = 0x20 = 32 (Real-time Interactive)
CS5  = 101000 = 0x28 = 40 (Signaling)
CS6  = 110000 = 0x30 = 48 (Network Control)
CS7  = 111000 = 0x38 = 56 (Reserved)

Assured Forwarding (AF) - 12 classes:
AF11 = 001010 = 0x0A = 10
AF12 = 001100 = 0x0C = 12
...
AF41 = 100010 = 0x22 = 34
AF42 = 100100 = 0x24 = 36
AF43 = 100110 = 0x26 = 38

Expedited Forwarding (EF) - Premium:
EF   = 101110 = 0x2E = 46 (VoIP, low-latency)
```

## Checking IPv6 Traffic Class

```bash
# View IPv6 Traffic Class in captured packets

sudo tcpdump -i eth0 -nn ip6 -v | grep "class\|tc 0x"

# Example output:
# IP6 (flowlabel 0x12345, hlim 64, next-header TCP (6) payload length: 40)
# 2001:db8::client.54321 > 2001:db8::server.443: ...
# ip6 class: 0x2e (DSCP EF)

# Use Wireshark display filter
# ipv6.tclass != 0  (show packets with non-zero Traffic Class)
# ipv6.dsfield.dscp == 46  (show EF-marked packets)
```

## Setting IPv6 Traffic Class with ip6tables

```bash
# Mark outbound IPv6 packets with DSCP values

# Mark VoIP traffic with EF (DSCP 46)
sudo ip6tables -t mangle -A OUTPUT \
  -p udp --dport 5060 \
  -j DSCP --set-dscp-class EF

# Mark video streaming with AF41
sudo ip6tables -t mangle -A OUTPUT \
  -p udp --dport 1234:1235 \
  -j DSCP --set-dscp-class AF41

# Mark SSH with AF31
sudo ip6tables -t mangle -A OUTPUT \
  -p tcp --dport 22 \
  -j DSCP --set-dscp-class AF31

# Mark ICMP6 with CS7 (highest priority - network control)
sudo ip6tables -t mangle -A OUTPUT \
  -p icmpv6 \
  -j DSCP --set-dscp 56
```

## Reading Traffic Class in Python

```python
# Inspect IPv6 Traffic Class field
import socket
import struct

def get_ipv6_traffic_class(raw_packet):
    """Extract Traffic Class from IPv6 header."""
    if len(raw_packet) < 4:
        return None

    # IPv6 header: version (4 bits) + traffic class (8 bits) + flow label (20 bits)
    first_word = struct.unpack('!I', raw_packet[:4])[0]
    traffic_class = (first_word >> 20) & 0xFF
    dscp = traffic_class >> 2
    ecn = traffic_class & 0x3

    return {
        'traffic_class': traffic_class,
        'dscp': dscp,
        'dscp_hex': hex(dscp),
        'ecn': ecn
    }

# Create raw socket to capture IPv6
sock = socket.socket(socket.AF_PACKET, socket.SOCK_RAW, socket.htons(0x86DD))
data, _ = sock.recvfrom(65535)
# Skip Ethernet header (14 bytes)
ipv6_packet = data[14:]
tc = get_ipv6_traffic_class(ipv6_packet)
print(f"Traffic Class: {tc}")
```

## ECN in IPv6 Traffic Class

```bash
# ECN (Explicit Congestion Notification) - bits 6-7 of Traffic Class

# ECN values:
# 00 = Not ECN-capable
# 01 = ECT(1) - ECN-capable transport
# 10 = ECT(0) - ECN-capable transport
# 11 = CE - Congestion Experienced

# Enable ECN on Linux for IPv6 connections
sudo sysctl -w net.ipv4.tcp_ecn=1  # Also applies to IPv6 TCP

# Check ECN is being used
sudo tcpdump -i eth0 ip6 -v | grep "ECN"
```

The IPv6 Traffic Class field provides identical QoS marking capability to IPv4's DSCP field, with the same 6-bit DSCP codepoint values mapping to Expedited Forwarding for voice, Assured Forwarding classes for data applications, and Class Selector values for network control traffic.
