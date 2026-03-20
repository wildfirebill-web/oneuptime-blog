# How to Use tshark for IPv6 Command-Line Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, tshark, Wireshark, Packet Analysis, Command Line, Network Diagnostics

Description: Use tshark, the command-line version of Wireshark, to capture and analyze IPv6 traffic with display filters, field extraction, and statistical analysis.

## Introduction

`tshark` is the command-line version of Wireshark that provides full dissection capabilities without a GUI. Unlike `tcpdump`, tshark uses Wireshark's display filter syntax and can extract specific fields, making it powerful for IPv6 analysis in scripts and automated workflows.

## Basic IPv6 Capture with tshark

```bash
# Capture all IPv6 traffic

tshark -i eth0 -f "ip6"

# Show only IPv6 frames with display filter
tshark -i eth0 -Y "ipv6"

# Capture with verbose decode
tshark -i eth0 -V -Y "ipv6"

# Capture ICMPv6 only
tshark -i eth0 -Y "icmpv6"

# Capture IPv6 HTTPS
tshark -i eth0 -Y "ipv6 && tcp.port == 443"

# Limit capture to 100 packets
tshark -i eth0 -c 100 -Y "ipv6"
```

## Extracting IPv6 Fields

```bash
# Extract source and destination IPv6 addresses
tshark -i eth0 -Y "ipv6" \
    -T fields -e ipv6.src -e ipv6.dst

# Extract with timestamp
tshark -i eth0 -Y "ipv6" \
    -T fields -e frame.time -e ipv6.src -e ipv6.dst -e ipv6.nxt

# Extract ICMPv6 type information
tshark -i eth0 -Y "icmpv6" \
    -T fields -e frame.time -e ipv6.src -e ipv6.dst -e icmpv6.type -e icmpv6.code

# CSV output for spreadsheet analysis
tshark -i eth0 -Y "ipv6" \
    -T fields -e frame.time -e ipv6.src -e ipv6.dst -e ipv6.plen \
    -E header=y -E separator=,

# Extract NDP neighbor solicitation targets
tshark -i eth0 -Y "icmpv6.type == 135" \
    -T fields -e ipv6.src -e icmpv6.nd.ns.target_address
```

## Reading and Filtering pcap Files

```bash
# Read existing pcap and show IPv6 traffic
tshark -r /tmp/capture.pcap -Y "ipv6"

# Filter by IPv6 address
tshark -r /tmp/capture.pcap -Y "ipv6.addr == 2001:db8::1"

# Filter ICMPv6 from pcap
tshark -r /tmp/capture.pcap -Y "icmpv6"

# Save filtered traffic to new pcap
tshark -r /tmp/original.pcap -Y "ipv6" -w /tmp/ipv6-only.pcap

# Count IPv6 packets by source address
tshark -r /tmp/capture.pcap -Y "ipv6" \
    -T fields -e ipv6.src | sort | uniq -c | sort -rn
```

## IPv6 Statistics with tshark

```bash
# IPv6 conversation statistics
tshark -r /tmp/capture.pcap -q -z conv,ipv6

# IPv6 endpoint statistics (bytes per address)
tshark -r /tmp/capture.pcap -q -z endpoints,ipv6

# ICMPv6 type distribution
tshark -r /tmp/capture.pcap -q -z io,stat,0,icmpv6.type==128,icmpv6.type==135,icmpv6.type==136

# Protocol hierarchy showing IPv6 breakdown
tshark -r /tmp/capture.pcap -q -z io,phs

# HTTP requests over IPv6
tshark -r /tmp/capture.pcap -Y "ipv6 && http.request" \
    -T fields -e frame.time -e ipv6.src -e http.host -e http.request.uri
```

## Automated IPv6 Analysis Script

```bash
#!/bin/bash
# tshark-ipv6-analysis.sh

PCAP="$1"
if [ -z "$PCAP" ]; then
    echo "Usage: $0 <pcap-file>"
    exit 1
fi

echo "=== IPv6 Traffic Analysis: $PCAP ==="

echo ""
echo "Total IPv6 packets:"
tshark -r "$PCAP" -Y "ipv6" -T fields -e frame.number 2>/dev/null | wc -l

echo ""
echo "Top IPv6 Source Addresses:"
tshark -r "$PCAP" -Y "ipv6" \
    -T fields -e ipv6.src 2>/dev/null | \
    sort | uniq -c | sort -rn | head -10

echo ""
echo "ICMPv6 Type Distribution:"
tshark -r "$PCAP" -Y "icmpv6" \
    -T fields -e icmpv6.type 2>/dev/null | \
    sort | uniq -c | sort -rn | while read count type; do
    case $type in
        128) name="Echo Request" ;;
        129) name="Echo Reply" ;;
        133) name="Router Solicitation" ;;
        134) name="Router Advertisement" ;;
        135) name="Neighbor Solicitation" ;;
        136) name="Neighbor Advertisement" ;;
        *) name="Type $type" ;;
    esac
    echo "  $count  $name"
done

echo ""
echo "IPv6 TCP Services:"
tshark -r "$PCAP" -Y "ipv6 && tcp" \
    -T fields -e tcp.dstport 2>/dev/null | \
    sort | uniq -c | sort -rn | head -10
```

## Live Capture to JSON for Processing

```bash
# Output IPv6 packets as JSON for further processing
tshark -i eth0 -Y "ipv6" -T json 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
for pkt in data:
    layers = pkt.get('_source', {}).get('layers', {})
    src = layers.get('ipv6', {}).get('ipv6.src', '')
    dst = layers.get('ipv6', {}).get('ipv6.dst', '')
    if src and dst:
        print(f'{src} -> {dst}')
"
```

## Conclusion

`tshark` brings Wireshark's powerful IPv6 dissection capabilities to the command line. Its display filter syntax (`ipv6`, `icmpv6.type`, `ipv6.addr`) is more expressive than BPF, and field extraction (`-T fields -e`) enables automated IPv6 traffic analysis in scripts. Use tshark for quick packet inspection, field extraction to CSV, and statistical reporting on IPv6 traffic patterns.
