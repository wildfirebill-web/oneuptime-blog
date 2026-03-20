# How to Use Capture Filters to Limit Traffic in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, BPF, Capture Filters, Networking, Packet Analysis

Description: Apply BPF-based capture filters in Wireshark before starting a capture to reduce stored data, focusing only on traffic relevant to your investigation.

Capture filters are applied before packets are stored - they reduce file size, improve performance, and keep your capture focused. Unlike display filters, they use BPF syntax (same as tcpdump) and cannot be changed during an active capture.

## Capture Filter vs Display Filter

```text
Feature              Capture Filter          Display Filter
-------------------  ----------------------  ----------------------
When applied?        Before storing packet   After packet is stored
Can change live?     No (stops capture)      Yes (real-time update)
Syntax               BPF (like tcpdump)      Wireshark display syntax
Performance impact   Reduces disk usage      No reduction in capture
Effect on data       Permanently excludes    Just hides, not deleted
```

## Setting a Capture Filter in Wireshark

```text
Method 1: Start screen
  Click on interface → click the small filter icon → type BPF filter

Method 2: Capture menu
  Capture → Options → Capture Filter field

Method 3: Double-click with filter
  Enter filter in field at top, then double-click interface
```

## Common Capture Filters

```bash
# These use BPF syntax, same as tcpdump

# Capture only traffic to/from a specific host

host 192.168.1.100

# Capture specific port
port 80
port 443

# HTTP and HTTPS only
port 80 or port 443
tcp port http or tcp port https

# Exclude SSH (don't capture your own session)
not port 22

# Traffic from a specific subnet
net 10.0.0.0/24
src net 192.168.1.0/24

# DNS only
udp port 53

# ICMP (ping) only
icmp
```

## Capture Filters for Specific Scenarios

```bash
# Web server traffic (HTTP + HTTPS + DNS)
port 80 or port 443 or udp port 53

# Database traffic (PostgreSQL + MySQL)
port 5432 or port 3306

# VPN diagnostics (WireGuard + OpenVPN)
udp port 51820 or udp port 1194

# All traffic except management protocols
not (port 22 or port 161 or port 162 or udp port 123)

# New TCP connections only (SYN packets)
tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & tcp-ack = 0
```

## Capture Filter for Specific IP Range

```bash
# Traffic from/to 192.168.1.0/24 subnet
net 192.168.1.0/24

# Traffic between two specific subnets
net 192.168.1.0/24 and net 10.0.0.0/8

# Exclude RFC 1918 private ranges (capture only internet traffic)
not (src net 10.0.0.0/8 or src net 172.16.0.0/12 or src net 192.168.0.0/16)
```

## Saving Capture Filters for Reuse

In Wireshark GUI:
```text
1. Enter the filter in the filter field
2. Click the "+" button next to the filter field
3. Name the filter
4. It appears in the saved filters list for future use

Access saved filters:
  Capture → Options → Manage Capture Filters
```

## Combine Capture and Display Filters

Use both for layered filtering:

```bash
# Capture filter: only HTTP/HTTPS (reduces capture size)
# In Wireshark capture filter: port 80 or port 443

# Then display filter: only traffic to specific IP
# In display filter bar: ip.dst == 203.0.113.10

# This combination stores only web traffic,
# then shows only traffic to a specific server from that web traffic
```

Capture filters are most valuable when analyzing high-traffic systems - without them, a busy production server generates gigabytes of capture data per minute, making analysis impractical.
