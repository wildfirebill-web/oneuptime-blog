# How to Apply Display Filters for IPv4 Traffic in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv4, Display Filters, Networking, Packet Analysis

Description: Use Wireshark display filters to show only IPv4 packets of interest from a larger capture, filtering by address, port, protocol, and field values.

Display filters in Wireshark are applied after capture — they show/hide packets without deleting them. Unlike capture filters, you can change display filters on the fly to explore different views of the same capture.

## Basic IPv4 Display Filters

Enter these in the filter bar at the top of the Wireshark window:

```wireshark
# Show only IPv4 packets
ip

# Show all IPv4 and IPv6
ip or ipv6

# Filter by source IP
ip.src == 192.168.1.100

# Filter by destination IP
ip.dst == 8.8.8.8

# Filter by source OR destination
ip.addr == 10.0.0.1

# Filter by subnet
ip.addr == 192.168.1.0/24
ip.src == 10.0.0.0/8
```

## Protocol Filters

```wireshark
# TCP only
tcp

# UDP only
udp

# ICMP only
icmp

# TCP on port 80
tcp.port == 80

# TCP destination port 443
tcp.dstport == 443

# Source port 22
tcp.srcport == 22

# DNS traffic
dns
# (equivalent to: udp.port == 53 or tcp.port == 53)
```

## Combining Filters

```wireshark
# Traffic between two specific IPs
ip.addr == 192.168.1.100 and ip.addr == 10.0.0.50

# HTTP/HTTPS traffic from a specific host
ip.src == 192.168.1.50 and (tcp.dstport == 80 or tcp.dstport == 443)

# All traffic EXCEPT SSH
not tcp.port == 22

# ICMP or TCP SYN packets
icmp or (tcp and tcp.flags.syn == 1 and tcp.flags.ack == 0)

# Traffic from subnet, excluding one IP
ip.src == 10.0.0.0/8 and not ip.src == 10.0.0.1
```

## TCP Flag Filters

```wireshark
# TCP SYN packets (new connections)
tcp.flags.syn == 1 and tcp.flags.ack == 0

# TCP SYN-ACK (connection accepted)
tcp.flags.syn == 1 and tcp.flags.ack == 1

# TCP RST (connection reset)
tcp.flags.reset == 1

# TCP FIN (connection closing)
tcp.flags.fin == 1

# All TCP connection initiation
tcp.flags.syn == 1
```

## IP Header Field Filters

```wireshark
# Filter by TTL value
ip.ttl < 10              # packets with low TTL (many hops used)
ip.ttl == 64             # Linux source TTL

# Filter by IP header length (packets with options)
ip.hdr_len > 20          # IP options present

# Filter by DSCP
ip.dsfield.dscp == 46    # Expedited Forwarding (EF)

# Fragmented packets
ip.flags.mf == 1         # More Fragments bit set
ip.frag_offset > 0       # Non-first fragment

# DF (Don't Fragment) bit set
ip.flags.df == 1
```

## Text Search in Packets

```wireshark
# Find packets containing a string in payload
frame contains "password"
frame contains "HTTP"

# Case-insensitive match
frame matches "(?i)login"

# Find HTTP GET requests
http.request.method == "GET"

# Find HTTP responses with error codes
http.response.code >= 400
```

## Save as New PCAP

After applying a display filter to find relevant packets, export them:

```
File → Export Specified Packets → check "Displayed" → Save
```

This creates a new PCAP containing only the filtered packets — useful for sharing specific traffic with colleagues.

Display filters are Wireshark's most powerful feature — they let you interactively explore a capture, progressively narrowing your focus without losing context of surrounding traffic.
