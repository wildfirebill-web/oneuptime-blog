# How to Detect IPv4 Fragmentation in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Fragmentation, Wireshark, Packet Analysis, MTU, Networking

Description: Use Wireshark display filters and analysis features to identify fragmented IPv4 packets, locate fragmentation points, and diagnose MTU-related issues.

## Introduction

Wireshark provides the clearest view of IPv4 fragmentation: it shows individual fragments, their offsets, whether reassembly succeeded, and flags the original packet that caused fragmentation. When you suspect an MTU issue or see unexplained packet loss, Wireshark fragmentation analysis reveals exactly where the problem is.

## Wireshark Display Filters for Fragmentation

```text
# Show fragmented packets (any fragment):

ip.flags.mf == 1 or ip.frag_offset > 0

# Show only first fragments (MF bit set, offset = 0):
ip.flags.mf == 1 and ip.frag_offset == 0

# Show non-first fragments (offset > 0):
ip.frag_offset > 0

# Show the Don't Fragment bit set:
ip.flags.df == 1

# Show large packets likely to cause fragmentation on standard Ethernet:
frame.len > 1500

# Show fragmented packets from specific IP:
(ip.flags.mf == 1 or ip.frag_offset > 0) and ip.src == 10.20.0.5

# Show ICMP fragmentation needed (PMTUD):
icmp.type == 3 and icmp.code == 4
```

## Understanding Wireshark Fragment Display

```text
In the packet list, fragmented packets look like:

Frame 100: [First fragment, MF bit set]
  Internet Protocol: src=10.0.0.1, dst=10.0.0.2
    Fragment Offset: 0
    Flags: More Fragment = 1, DF = 0
    Total Length: 1500

Frame 101: [Second fragment, last]
  Internet Protocol: src=10.0.0.1, dst=10.0.0.2
    Fragment Offset: 185 (185 × 8 = 1480 bytes)
    Flags: More Fragment = 0, DF = 0

Frame 102: [Reassembled, Wireshark shows the full UDP/TCP data]
  [Reassembled from frames 100, 101]
```

## Find Where Fragmentation Occurs

```bash
# If you capture at both sender and receiver:
# Sender capture: large unfragmented packet (if sender MTU is large)
# Receiver capture: fragmented packets
# → Fragmentation occurring at router between them

# If you capture at router ingress and egress:
# Ingress: large unfragmented packet
# Egress: fragmented packets
# → This router is fragmenting (its outbound link has lower MTU)

# To find fragmentation point:
# 1. Capture at source: no fragmentation?
# 2. Capture at intermediate hop: fragmentation starts here?
# 3. The link after the first point where fragmentation appears = bottleneck
```

## Test Fragmentation with tcpdump

```bash
# Generate fragmented traffic for testing:
# ping with packet size > 1472 causes fragmentation:
ping -s 2000 -c 5 10.20.0.5

# Capture the fragments:
tcpdump -i eth0 -n -v 'host 10.20.0.5 and (ip[6:2] & 0x3fff) != 0'

# Expected output for fragmented ping:
# IP 10.20.0.1 > 10.20.0.5: ICMP echo request, id 1234, seq 1, length 2008
#   frag 54321:0+ (offset=0, MF bit set - first fragment)
# IP 10.20.0.1 > 10.20.0.5: ICMP (frag 54321:540)
#   frag 54321:1480 (offset=1480, last fragment)
```

## Wireshark Expert Information for Fragmentation

```text
In Wireshark:
  Analyze → Expert Information

Look for:
  - "IPv4 fragment" - notes about fragmented packets
  - "Reassembled in frame N" - where reassembly completed
  - "IPv4 fragment overlaps previous fragment" - fragment attack or bug
  - "IPv4 datagrams must not be fragmented" - DF bit set on fragmented packet

Statistics → IPv4 Statistics → All Addresses
Shows: packet count, bytes, fragments per source/destination
```

## Identify MTU Black Holes

```bash
# MTU black hole: host sends packets with DF bit set
# Router on path needs to fragment but CAN'T (DF bit set)
# Router should send ICMP Fragmentation Needed (type 3, code 4)
# But if ICMP is blocked: packet is silently dropped

# Detect black holes in Wireshark:
# Filter: tcp and frame.len > 1400 and ip.flags.df == 1
# If you see SYN packets succeeding but data transfer hanging:
# → Large TCP packets with DF bit are being dropped by a black hole router

# Also watch for:
# tcp.analysis.retransmission   (retransmits of specific size packets)
# Correlate with frame size: if only large frames are retransmitted → black hole
```

## Conclusion

Wireshark fragmentation analysis uses two primary filters: `ip.flags.mf == 1 or ip.frag_offset > 0` to find all fragments, and `icmp.type == 3 and icmp.code == 4` to find PMTUD fragmentation needed messages. When fragmentation appears in a capture, the router between capture points is fragmenting (its outbound MTU is smaller than the incoming packet). MTU black holes are detected by large packets with DF bit being silently dropped - watch for retransmissions only affecting large frames. Fix by reducing MTU, enabling MSS clamping, or ensuring ICMP fragmentation needed messages are not blocked.
