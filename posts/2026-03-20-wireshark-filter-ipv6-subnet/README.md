# How to Filter IPv6 Packets by Subnet in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv6, Subnets, Display Filters, Packet Analysis, CIDR

Description: A guide to filtering Wireshark captures to show only traffic within or between specific IPv6 subnets using CIDR notation and display filter syntax.

Wireshark supports CIDR-notation subnet filtering for IPv6, making it possible to quickly isolate traffic within a specific /48, /64, or other prefix length. This is essential for analyzing inter-subnet traffic flow and debugging routing.

## CIDR Subnet Filters

Wireshark's display filter language supports the `/prefix-length` notation for IPv6:

```wireshark
# All traffic within the 2001:db8:1::/64 subnet (source OR destination)
ipv6.addr == 2001:db8:1::/64

# All traffic with source in the subnet
ipv6.src == 2001:db8:1::/64

# All traffic with destination in the subnet
ipv6.dst == 2001:db8:1::/64
```

## Filter Traffic Between Two Subnets

```wireshark
# Traffic flowing from subnet A to subnet B (unidirectional)
ipv6.src == 2001:db8:clients::/64 && ipv6.dst == 2001:db8:servers::/64

# Traffic between subnet A and subnet B (bidirectional)
(ipv6.src == 2001:db8:clients::/64 && ipv6.dst == 2001:db8:servers::/64)
||
(ipv6.src == 2001:db8:servers::/64 && ipv6.dst == 2001:db8:clients::/64)

# Shorthand: any traffic where EITHER endpoint is in the subnet
ipv6.addr == 2001:db8:clients::/64 && ipv6.addr == 2001:db8:servers::/64
```

## Filter Intra-Subnet Traffic

To see only traffic where both source and destination are in the same /64:

```wireshark
# Both source and destination in the servers subnet
ipv6.src == 2001:db8:servers::/64 && ipv6.dst == 2001:db8:servers::/64
```

## Filter by /48 Site Prefix

```wireshark
# All traffic within a site's /48 block
ipv6.addr == 2001:db8:site1::/48

# Traffic crossing between two sites
(ipv6.src == 2001:db8:site1::/48 && ipv6.dst == 2001:db8:site2::/48)
||
(ipv6.src == 2001:db8:site2::/48 && ipv6.dst == 2001:db8:site1::/48)
```

## Exclude Specific Subnets

```wireshark
# All IPv6 traffic EXCEPT loopback
ipv6 && !(ipv6.addr == ::1/128)

# All IPv6 except link-local
ipv6 && !(ipv6.addr == fe80::/10)

# All IPv6 except multicast
ipv6 && !(ipv6.addr == ff00::/8)

# All IPv6 except internal network
ipv6 && !(ipv6.addr == 2001:db8:internal::/48)
```

## Subnet Filters with Protocol Restrictions

```wireshark
# HTTP traffic to/from the web server subnet
ipv6.addr == 2001:db8:web::/64 && http

# DNS queries from clients subnet to DNS servers subnet
ipv6.src == 2001:db8:clients::/64 && ipv6.dst == 2001:db8:dns::/64 && dns

# All HTTPS traffic crossing the DMZ subnet boundary
ipv6.addr == 2001:db8:dmz::/64 && tcp.port == 443
```

## Using tshark for Command-Line Subnet Filtering

```bash
# Filter a pcap file for traffic within a subnet
tshark -r capture.pcap -Y "ipv6.addr == 2001:db8:1::/64"

# Count packets per subnet pair
tshark -r capture.pcap -Y "ipv6" \
  -T fields -e ipv6.src -e ipv6.dst | \
  awk '{print $1, $2}' | sort | uniq -c | sort -rn | head -20

# Filter and save subnet-specific traffic
tshark -r capture.pcap \
  -Y "ipv6.src == 2001:db8:clients::/64 && ipv6.dst == 2001:db8:servers::/64" \
  -w clients-to-servers.pcap
```

## Practical Use Cases

| Filter | Use Case |
|---|---|
| `ipv6.src == 2001:db8:clients::/64` | Audit client traffic |
| `ipv6.addr == 2001:db8:dmz::/64` | Monitor DMZ activity |
| `!(ipv6.addr == 2001:db8::/32)` | Find traffic from unknown prefixes |
| `ipv6.src == fe80::/10` | Analyze link-local (NDP) traffic |

IPv6 subnet filtering in Wireshark is one of the most powerful tools for understanding traffic patterns at the network segment level, making it indispensable for firewall rule validation and inter-subnet routing analysis.
