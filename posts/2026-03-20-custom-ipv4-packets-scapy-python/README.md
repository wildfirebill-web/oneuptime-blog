# How to Craft Custom IPv4 Packets Using Scapy in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Scapy, IPv4, Packet Crafting, Networking, Security

Description: Learn how to craft and send custom IPv4 packets using Scapy, including setting IP headers, payloads, and sending packets at the raw socket level.

## What Is Scapy?

Scapy is a Python library for creating, sending, sniffing, and dissecting network packets. It lets you construct packets layer by layer—Ethernet, IP, TCP, UDP, ICMP—and send them directly on the wire.

## Installation

```bash
pip install scapy

# Root/sudo privileges are required for raw socket operations
```

## Building a Basic IPv4 Packet

```python
from scapy.all import IP, TCP, UDP, ICMP, send, sr1

# Create an IPv4 packet with a TCP layer
# IP() constructs the IPv4 header; TCP() adds the TCP header
packet = IP(dst="192.168.1.1") / TCP(dport=80, flags="S")

# Display the packet structure
packet.show()
```

## Setting IPv4 Header Fields

```python
from scapy.all import IP, TCP

# Customize IPv4 header fields
ip_header = IP(
    src="192.168.1.50",    # Source IP (can spoof; requires root)
    dst="192.168.1.1",     # Destination IP
    ttl=64,                # Time to Live
    tos=0,                 # Type of Service / DSCP
    id=1234,               # IP Identification field
    flags="DF",            # Don't Fragment flag
)

# Add a TCP layer
tcp_header = TCP(
    sport=12345,           # Source port
    dport=80,              # Destination port
    flags="S",             # SYN flag
    seq=1000,              # Initial sequence number
    window=65535,          # Window size
)

# Stack layers using the / operator
packet = ip_header / tcp_header
packet.show2()  # show2() computes and shows checksums
```

## Sending a Packet

```python
from scapy.all import IP, TCP, send, sr1

# send() sends at Layer 3 (no response handling)
# sr1() sends and receives exactly one response (send/receive 1)
packet = IP(dst="8.8.8.8") / TCP(dport=53, flags="S")

# Send without waiting for response
send(packet, verbose=False)

# Send and wait for a single reply (timeout=2 seconds)
reply = sr1(packet, timeout=2, verbose=False)
if reply:
    print(f"Got reply from {reply[IP].src}: flags={reply[TCP].flags}")
else:
    print("No reply received")
```

## Crafting a Custom UDP Packet with Payload

```python
from scapy.all import IP, UDP, Raw, send

# Build a UDP packet with a raw text payload
packet = (
    IP(dst="192.168.1.100") /
    UDP(sport=9999, dport=9001) /
    Raw(load=b"Hello from Scapy!")
)

send(packet, verbose=True)
```

## Crafting an ICMP Ping

```python
from scapy.all import IP, ICMP, sr1

# Craft an ICMP Echo Request (ping)
ping = IP(dst="8.8.8.8") / ICMP()

reply = sr1(ping, timeout=2, verbose=False)
if reply:
    print(f"Ping reply from {reply[IP].src}, TTL={reply[IP].ttl}")
```

## Sending Multiple Packets

```python
from scapy.all import IP, TCP, sendp, Ether, RandShort

# Send a SYN scan to ports 20-25
packets = [
    IP(dst="192.168.1.1") / TCP(dport=port, flags="S", sport=RandShort())
    for port in range(20, 26)
]

# send() accepts a list
from scapy.all import send
send(packets, verbose=False)
```

## Important Notes

- Raw packet sending requires root/administrator privileges
- Spoofing source IPs is illegal on networks you don't own
- Use Scapy only on networks you have explicit permission to test

## Conclusion

Scapy makes IPv4 packet crafting intuitive with its layered `IP() / TCP() / Raw()` syntax. It's invaluable for network security testing, protocol research, and building custom network probes. Always operate within authorized environments.
