# How to Filter Wireshark Traffic by IPv4 Address Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv4, Display Filters, Subnets, Networking, Analysis

Description: Filter Wireshark captures to show only traffic from or to specific IPv4 address ranges, subnets, or CIDR blocks using display filter syntax.

Filtering by address range lets you focus on specific network segments, investigate a particular subnet, or exclude internal traffic to analyze only external connections.

## Filter by Single IP Address

```wireshark
# Traffic to or from a specific IP

ip.addr == 192.168.1.100

# Traffic FROM a specific IP (source)
ip.src == 192.168.1.100

# Traffic TO a specific IP (destination)
ip.dst == 192.168.1.100
```

## Filter by Subnet (CIDR Range)

```wireshark
# Traffic to or from any IP in 192.168.1.0/24
ip.addr == 192.168.1.0/24

# Traffic FROM a /24 subnet
ip.src == 192.168.1.0/24

# Traffic TO a /24 subnet
ip.dst == 192.168.1.0/24

# Traffic from a /16 network
ip.src == 10.0.0.0/16

# Traffic from the entire 10.x.x.x range
ip.src == 10.0.0.0/8
```

## Filter by Multiple Address Ranges

```wireshark
# Traffic from RFC 1918 private ranges (all internal)
ip.src == 10.0.0.0/8 or ip.src == 172.16.0.0/12 or ip.src == 192.168.0.0/16

# Traffic from either of two subnets
ip.addr == 192.168.1.0/24 or ip.addr == 10.100.0.0/24

# Traffic between two specific subnets
ip.src == 192.168.1.0/24 and ip.dst == 10.100.0.0/24
```

## Exclude Address Ranges

```wireshark
# Show only external (non-private) traffic
not (ip.addr == 10.0.0.0/8 or ip.addr == 172.16.0.0/12 or ip.addr == 192.168.0.0/16)

# Exclude a specific subnet
not ip.addr == 192.168.1.0/24

# Exclude loopback
not ip.addr == 127.0.0.0/8

# Show all traffic except from internal management network
not ip.src == 10.100.0.0/24
```

## Combine with Protocol Filters

```wireshark
# HTTP traffic from a specific subnet
http and ip.src == 192.168.1.0/24

# DNS queries from a specific range
dns and ip.src == 10.0.0.0/8

# TCP connections to your web servers
tcp.port == 443 and ip.dst == 203.0.113.0/24

# ICMP from external (non-private) sources
icmp and not ip.src == 192.168.0.0/16 and not ip.src == 10.0.0.0/8
```

## IP Range Filters with Other Conditions

```wireshark
# Large packets from a specific subnet (potential large transfers)
ip.src == 10.0.0.0/8 and frame.len > 1400

# New connections from external IPs
tcp.flags.syn == 1 and tcp.flags.ack == 0 and
  not (ip.src == 10.0.0.0/8 or ip.src == 192.168.0.0/16)

# Errors from internal hosts
(tcp.analysis.retransmission or dns.flags.rcode != 0) and
  ip.src == 192.168.1.0/24
```

## Find Traffic Between Two Hosts in Different Subnets

```wireshark
# Traffic between 192.168.1.x and 10.200.x.x
(ip.src == 192.168.1.0/24 and ip.dst == 10.200.0.0/16) or
(ip.src == 10.200.0.0/16 and ip.dst == 192.168.1.0/24)
```

## Command-Line Filtering by Range with tshark

```bash
# Export traffic from a subnet using tshark
tshark -r capture.pcap \
  -Y 'ip.src == 192.168.1.0/24' \
  -w /tmp/subnet-traffic.pcap

# Count packets by source /24 subnet
tshark -r capture.pcap -T fields -e ip.src \
  | awk -F. '{print $1"."$2"."$3".0/24"}' \
  | sort | uniq -c | sort -rn | head -10
```

Address range filtering is especially valuable in enterprise environments where you need to investigate a specific segment (DMZ, production, dev) without drowning in traffic from other parts of the network.
