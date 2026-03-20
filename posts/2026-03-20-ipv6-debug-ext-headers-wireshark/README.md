# How to Debug Extension Header Issues with Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Wireshark, Extension Headers, Debugging, Packet Analysis

Description: Use Wireshark to capture, filter, and analyze IPv6 extension headers, decode their contents, and diagnose extension header-related connectivity problems.

## Introduction

Wireshark provides a rich graphical and filter-based interface for analyzing IPv6 extension headers that goes beyond what tcpdump can display. Its dissectors automatically decode each extension header type, making it easy to inspect routing headers, fragment headers, and options. This guide covers the most useful filters and techniques for extension header debugging.

## Key Wireshark Display Filters

```text
# Capture all IPv6 traffic

ip.version == 6
ipv6

# Filter by Next Header (extension header type)
ipv6.nxt == 44   # Fragment Header
ipv6.nxt == 43   # Routing Header
ipv6.nxt == 0    # Hop-by-Hop Options
ipv6.nxt == 51   # Authentication Header
ipv6.nxt == 50   # ESP

# Filter IPv6 packets that HAVE extension headers
# (any packet where the base Next Header != upper-layer protocol)
not (ipv6.nxt == 6 or ipv6.nxt == 17 or ipv6.nxt == 58 or ipv6.nxt == 59)

# Find fragmented packets
ipv6.fragment

# Find the first fragment of a fragmented sequence
ipv6.fragment.offset == 0 and ipv6.fragment.more == 1

# Find the last fragment
ipv6.fragment.more == 0 and ipv6.fragment.offset != 0

# Filter by Fragment ID
ipv6.fragment.id == 0x12345678

# Filter by Flow Label
ipv6.flow == 0x2a3b4

# Filter Routing Header by type
ipv6.routing.type == 2    # Type 2 (Mobile IPv6)
ipv6.routing.type == 0    # Type 0 (deprecated, security risk)
ipv6.routing.type == 4    # Type 4 (Segment Routing)
```

## Wireshark Capture Filter (BPF) for Extension Headers

These go in the "Capture filter" field to filter at capture time:

```text
# Capture packets with Fragment Header (Next Header byte at offset 6)
ip6[6] == 44

# Capture packets with Hop-by-Hop (Next Header byte at offset 6)
ip6[6] == 0

# Capture packets with any extension header (not direct to upper layer)
not (ip6[6] == 6 or ip6[6] == 17 or ip6[6] == 58 or ip6[6] == 59)

# Capture fragments
ip6[6] == 44 and (ip6[48] & 0x01) == 1  # Fragment with M=1
```

## Setting Up a Wireshark Capture

```bash
# Capture IPv6 extension headers to a file for Wireshark analysis
sudo tcpdump -i eth0 -w /tmp/ext-headers.pcap \
    "ip6[6] == 44 or ip6[6] == 43 or ip6[6] == 0 or ip6[6] == 51"

# Open in Wireshark
wireshark /tmp/ext-headers.pcap &

# Or use tshark (Wireshark command line) for scripted analysis
tshark -r /tmp/ext-headers.pcap -Y "ipv6.nxt == 44" \
    -T fields -e frame.number -e ipv6.src -e ipv6.dst \
    -e ipv6.fragment.id -e ipv6.fragment.offset \
    -e ipv6.fragment.more
```

## Analyzing Fragment Reassembly

Wireshark can automatically reassemble fragments and show the reassembled packet:

```bash
# tshark: show fragment reassembly information
tshark -r capture.pcap -Y "ipv6.fragment" \
    -T fields \
    -e frame.number \
    -e ipv6.src \
    -e ipv6.dst \
    -e ipv6.fragment.id \
    -e ipv6.fragment.offset \
    -e ipv6.fragment.more \
    -e ipv6.fragment.overlap \
    -e ipv6.fragments \
    -e ipv6.reassembled_length

# Enable IPv6 reassembly in tshark
tshark -r capture.pcap \
    -o "ipv6.reassemble_fragments:TRUE" \
    -Y "ip.reassembled_in != 0"  # Show reassembled packets
```

## Debugging Routing Header Issues

```bash
# Find all packets with Routing Headers
tshark -r capture.pcap -Y "ipv6.routing" \
    -T fields \
    -e frame.number \
    -e ipv6.src \
    -e ipv6.dst \
    -e ipv6.routing.type \
    -e ipv6.routing.segleft

# Check for deprecated Type 0 routing headers
tshark -r capture.pcap -Y "ipv6.routing.type == 0" \
    -T text
# These should not exist in production traffic
```

## Wireshark Coloring Rules for Extension Headers

Add these coloring rules in Edit → Coloring Rules to visually highlight extension headers:

```text
Rule name: IPv6 Fragment
Filter: ipv6.fragment
Background: Orange
Foreground: Black

Rule name: IPv6 with Extension Headers
Filter: (ipv6) and not (ipv6.nxt == 6 or ipv6.nxt == 17 or ipv6.nxt == 58)
Background: Light blue
Foreground: Black

Rule name: Deprecated RH0
Filter: ipv6.routing.type == 0
Background: Red
Foreground: White
```

## Diagnosing Connectivity Issues from Extension Header Drops

```bash
# Scenario: HTTPS connections work for small data but fail for large transfers
# Suspect: Fragment Header being dropped

# Step 1: Capture traffic during the failure
sudo tcpdump -i eth0 -w /tmp/debug.pcap host 2001:db8::server

# Step 2: Open in Wireshark and look for ICMPv6 Packet Too Big
tshark -r /tmp/debug.pcap -Y "icmpv6.type == 2" \
    -T fields -e ipv6.src -e ipv6.dst \
    -e icmpv6.mtu

# Step 3: Check if fragments are being created but not received
tshark -r /tmp/debug.pcap -Y "ipv6.fragment"
# Count outgoing fragments vs reassembled packets

# Step 4: Check for ICMPv6 Time Exceeded (fragment reassembly timeout)
tshark -r /tmp/debug.pcap -Y "icmpv6.type == 3 and icmpv6.code == 1"
```

## Conclusion

Wireshark's IPv6 dissectors provide detailed visibility into every extension header type, with automatic fragment reassembly and rich filtering capabilities. The display filters `ipv6.nxt == 44` (fragment), `ipv6.routing.type` (routing type), and `ipv6.fragment` are the most commonly needed for extension header debugging. When diagnosing mysterious IPv6 connectivity failures, always look for ICMPv6 Packet Too Big messages (type 2) and fragment header drops, as these are the most common causes of "works with small packets but fails with large data" symptoms.
