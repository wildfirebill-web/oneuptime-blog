# How to Identify the Protocol Field in IPv4 Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, TCP/IP, Protocol Numbers, Packet Analysis

Description: The Protocol field in the IPv4 header is an 8-bit value that identifies which upper-layer protocol (TCP, UDP, ICMP, etc.) is carried in the datagram's payload.

## What Is the Protocol Field?

The Protocol field is an 8-bit value located at byte offset 9 in the IPv4 header. It tells the receiving host which protocol should process the payload after the IP layer strips its header. The values are assigned by IANA and documented in RFC 790 and its successors.

## Common Protocol Numbers

| Number | Protocol | Description |
|--------|----------|-------------|
| 1  | ICMP   | Internet Control Message Protocol |
| 2  | IGMP   | Internet Group Management Protocol |
| 6  | TCP    | Transmission Control Protocol |
| 17 | UDP    | User Datagram Protocol |
| 41 | IPv6   | IPv6 encapsulated in IPv4 |
| 47 | GRE    | Generic Routing Encapsulation |
| 50 | ESP    | IPsec Encapsulating Security Payload |
| 51 | AH     | IPsec Authentication Header |
| 89 | OSPF   | Open Shortest Path First |

## Reading the Protocol Field with Python

```python
import socket
import struct

def parse_ipv4_protocol(raw_packet: bytes) -> int:
    """
    Parse the Protocol field from a raw IPv4 packet.
    Assumes the packet starts at the IP header (no Ethernet frame).
    """
    # IP header protocol field is at byte offset 9
    protocol = raw_packet[9]
    return protocol

# Map protocol numbers to names

PROTO_NAMES = {1: "ICMP", 6: "TCP", 17: "UDP", 47: "GRE", 89: "OSPF"}

# Example: reading from a raw socket
with socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_TCP) as s:
    s.settimeout(5)
    try:
        data, addr = s.recvfrom(65535)
        proto = parse_ipv4_protocol(data)
        print(f"From {addr[0]}: Protocol={proto} ({PROTO_NAMES.get(proto, 'Unknown')})")
    except socket.timeout:
        print("No packet received")
```

## Filtering by Protocol with tcpdump

```bash
# Capture only ICMP packets
tcpdump -i eth0 'ip proto 1'

# Capture only UDP packets
tcpdump -i eth0 'ip proto 17'

# Capture GRE tunnel traffic
tcpdump -i eth0 'ip proto 47'

# Display protocol number for each packet
tcpdump -i eth0 -v | grep proto
```

## Filtering with iptables by Protocol

```bash
# Allow incoming TCP traffic on port 443
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block all ICMP (protocol 1)
iptables -A INPUT -p icmp -j DROP

# Allow GRE (protocol 47) for VPN tunnels
iptables -A INPUT -p gre -j ACCEPT
```

## Looking Up All Protocol Numbers

The full list of IANA-assigned protocol numbers is in `/etc/protocols` on Linux:

```bash
# View the protocols database
cat /etc/protocols

# Look up a specific protocol by number
grep "^[^#]" /etc/protocols | awk '$2 == 6'
```

## Key Takeaways

- The Protocol field at byte 9 of the IPv4 header identifies the encapsulated upper-layer protocol.
- Common values are 1 (ICMP), 6 (TCP), and 17 (UDP).
- Use `/etc/protocols` or IANA's registry for a complete mapping.
- Firewall rules and packet filters rely heavily on this field for protocol-based filtering.
