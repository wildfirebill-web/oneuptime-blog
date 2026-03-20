# How to Capture and Filter UDP Traffic in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Wireshark, Packet Capture, Network Analysis, Debugging

Description: Use Wireshark capture and display filters to isolate and analyze UDP traffic, decode known protocols, and extract UDP statistics.

## Introduction

Wireshark provides the most detailed view of UDP traffic available: every packet with full header decoding, payload display, timing, and protocol dissection for known UDP protocols like DNS, DHCP, and NTP. Effective Wireshark use requires knowing both capture filters (applied before capture to limit disk usage) and display filters (applied after capture for analysis).

## Capture Filters for UDP

```
Capture filters (BPF syntax, set before capture starts):

# All UDP traffic:
udp

# UDP on specific port:
udp port 5000

# UDP from specific host:
udp and host 10.20.0.5

# UDP on source or destination port:
udp src port 5000
udp dst port 5000

# UDP not on common ports (exclude DNS, NTP noise):
udp and not port 53 and not port 123

# Large UDP packets (potential fragmentation):
udp and greater 1400
```

## Display Filters for UDP

```
Display filters (applied to already-captured traffic):

# Show only UDP:
udp

# Specific UDP port:
udp.port == 5000
udp.srcport == 5000
udp.dstport == 5000

# UDP length filtering:
udp.length > 1400   # Large UDP packets

# UDP checksum errors:
udp.checksum_bad == true

# Combined filters:
udp.port == 5000 and ip.addr == 10.20.0.5

# Show UDP traffic excluding DNS:
udp and not dns

# Find UDP port unreachable (ICMP response to closed UDP port):
icmp.type == 3 and icmp.code == 3
```

## Protocol Dissection for Known UDP Protocols

```
Wireshark automatically dissects these UDP protocols:

DNS (port 53):
  - filter: dns
  - Shows: query type, domain, response code
  - Statistics → DNS to see response time distribution

DHCP (ports 67/68):
  - filter: dhcp or bootp
  - Shows: message type, client/server interaction, options

NTP (port 123):
  - filter: ntp
  - Shows: stratum, reference timestamp, offset

SNMP (port 161):
  - filter: snmp
  - Shows: community string, OID, value

RTP (dynamic ports):
  - Telephony → RTP → Show All Streams
  - Shows: payload type, sequence numbers, timestamps
  - Analyze → Expert Information for RTP issues

TFTP (port 69):
  - filter: tftp
  - Shows: file transfer operations
```

## Decode UDP as Specific Protocol

```bash
# Wireshark may not know which protocol is on a custom port
# Force decode as a specific protocol:

# In Wireshark GUI:
# Right-click a UDP packet → Decode As...
# Select the UDP port → Set protocol to desired decoder

# From command line with tshark:
# Decode port 5555 as DNS:
tshark -r capture.pcap -d udp.port==5555,dns

# Decode as RTP:
tshark -r capture.pcap -d udp.port==5004,rtp
```

## UDP Statistics

```bash
# In Wireshark GUI:
# Statistics → Conversations → UDP tab
# Shows: address pairs, packet counts, bytes, duration

# Statistics → Endpoints → UDP tab
# Shows: top UDP talkers

# From command line with tshark:
# Get UDP conversation stats:
tshark -r capture.pcap -q -z conv,udp

# Top UDP source/destination pairs:
tshark -r capture.pcap -T fields -e ip.src -e udp.srcport -e ip.dst -e udp.dstport \
  | sort | uniq -c | sort -rn | head -20

# UDP packet size distribution:
tshark -r capture.pcap -T fields -e udp.length | sort -n | uniq -c
```

## Export UDP Payload

```bash
# Extract UDP payload data from a capture:
# In Wireshark: right-click a UDP packet body → Export Packet Bytes

# From tshark command line:
# Export all UDP payloads to stdout (hex):
tshark -r capture.pcap -T fields -e data.data 'udp and dst port 5000'

# Save all UDP packets to separate files:
tcpdump -r capture.pcap -w - 'udp port 5000' | \
  tshark -r - -w /tmp/udp_5000.pcap

# Extract DNS query names from capture:
tshark -r capture.pcap -Y 'dns.qry.type == 1' -T fields -e dns.qry.name | sort -u
```

## Conclusion

Wireshark's UDP analysis starts with a capture filter to limit data volume, then display filters to isolate specific flows. For known protocols like DNS, DHCP, and RTP, Wireshark decodes the payload automatically. For custom protocols, use "Decode As" to force the right dissector. Use `tshark` for command-line analysis and automation. The conversations/endpoints statistics window gives a quick overview of which UDP flows are using the most bandwidth.
