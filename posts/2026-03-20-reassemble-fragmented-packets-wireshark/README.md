# How to Reassemble Fragmented IPv4 Packets in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Fragmentation, Wireshark, Packet Analysis, Reassembly, Networking

Description: Use Wireshark's automatic fragment reassembly to analyze the original unfragmented data, view complete payloads, and troubleshoot fragmentation issues in packet captures.

## Introduction

Wireshark automatically reassembles IPv4 fragments by default. When you capture fragmented traffic, Wireshark displays individual fragments as they arrive on the wire, and also shows a "reassembled" view at the last fragment with the complete original payload. Understanding how to read Wireshark's fragment display and how to work with fragment captures is essential for analyzing protocol behavior in fragmented environments.

## How Wireshark Displays Fragments

```
In the packet list, fragmented packets appear as:

Frame 100: IP fragment (offset=0, MF=1)
  Source: 10.0.0.1
  Proto: UDP (but can't decode yet - waiting for all fragments)
  Fragment Offset: 0
  More Fragments: Yes

Frame 101: IP fragment (offset=1480)
  Source: 10.0.0.1
  Fragment Offset: 1480
  More Fragments: No

Frame 101 (expanded in detail pane):
  [Reassembled IPv4 in frame: 101]      ← Reassembly completed here
  UDP, Source Port: 5000
  Data (2960 bytes)                     ← Full original payload visible
```

## Configure Wireshark Fragment Reassembly

```
In Wireshark:
  Edit → Preferences → Protocols → IPv4

  ☑ Reassemble fragmented IPv4 datagrams
  (This is enabled by default)

  If disabled: you only see individual fragments
  If enabled: you see both fragments AND the reassembled packet
```

## Wireshark Display Filters for Fragments

```
# Show all fragment frames:
ip.flags.mf == 1 or ip.frag_offset > 0

# Show first fragments only:
ip.flags.mf == 1 and ip.frag_offset == 0

# Show last fragments (where reassembly is displayed):
ip.flags.mf == 0 and ip.frag_offset > 0

# Show the reassembled view (only visible in last fragment):
_ws.col.protocol == "UDP" and ip.frag_offset > 0
# (After reassembly, last fragment shows actual protocol)

# Find incomplete fragment sets (missing fragments):
# These don't appear automatically - need to check timing
```

## Analyze Reassembled Data

```bash
# Capture fragmented traffic:
# Generate fragmented UDP packets:
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
# Send large payload that will be fragmented:
s.sendto(b'X' * 3000, ('10.20.0.5', 5000))
s.close()
"

# In Wireshark:
# 1. Open capture
# 2. Find the last fragment (no MF bit, offset > 0)
# 3. Click on it
# 4. In detail pane: see full reassembled UDP payload
# 5. Right-click → Copy → All Visible Items to export
```

## Export Reassembled Payload

```bash
# Export fragment payload from command line with tshark:

# Show reassembled data for fragmented UDP:
tshark -r capture.pcap \
  -Y "ip.flags.mf == 0 and ip.frag_offset > 0" \
  -T fields -e data.data

# Export reassembled UDP payloads as binary:
tshark -r capture.pcap \
  -Y "udp" \
  -T fields -e data | xxd -r -p > /tmp/reassembled_payload.bin
# Note: works on reassembled frames

# Decode reassembled DNS (fragmented):
tshark -r capture.pcap \
  -Y "dns and (ip.frag_offset > 0 or ip.flags.mf == 1)" \
  -V
```

## Identify Missing Fragments

```bash
# Missing fragments prevent reassembly:
# Wireshark marks incomplete reassembly in Expert Information

# In Wireshark:
# Analyze → Expert Information
# Look for: "IPv4 Fragment" warnings or reassembly failures

# Using tshark to find fragmented packets missing their pairs:
tshark -r capture.pcap -T fields \
  -e frame.number -e ip.id -e ip.frag_offset -e ip.flags.mf \
  -Y "ip.flags.mf == 1 or ip.frag_offset > 0" | \
  awk '{print $2, $3, $4}' | sort | head -30
# Group by ip.id to see which fragment sets are complete

# Check for timeout-related drops:
# If you see fragment 1 and 2 but never fragment 3: packet loss
```

## tshark Fragment Statistics

```bash
# Count fragment events in a capture:
echo "Fragment Statistics:"
echo "First fragments:"
tshark -r capture.pcap -Y "ip.flags.mf == 1 and ip.frag_offset == 0" 2>/dev/null \
  | wc -l

echo "All fragments:"
tshark -r capture.pcap -Y "ip.flags.mf == 1 or ip.frag_offset > 0" 2>/dev/null \
  | wc -l

echo "Last fragments (reassembly point):"
tshark -r capture.pcap -Y "ip.flags.mf == 0 and ip.frag_offset > 0" 2>/dev/null \
  | wc -l

echo "Unique fragment IDs:"
tshark -r capture.pcap -Y "ip.flags.mf == 1 or ip.frag_offset > 0" \
  -T fields -e ip.id 2>/dev/null | sort -u | wc -l
```

## Conclusion

Wireshark handles IPv4 fragment reassembly automatically, showing both the individual wire-level fragments and the reassembled payload at the last fragment. Use display filter `ip.flags.mf == 1 or ip.frag_offset > 0` to find all fragment frames. The last fragment frame (`ip.flags.mf == 0 and ip.frag_offset > 0`) contains the full reassembled protocol data. Expert Information reveals reassembly failures. For command-line work, `tshark` provides the same reassembly capability for scripted analysis of fragmented captures.
