# How to Use ICMP Timestamp Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, Networking, Timestamps, IPv4, Time Synchronization, Security

Description: Understand ICMP Timestamp Request and Reply messages, how they measure one-way network delay, and why they are typically disabled for security reasons.

## Introduction

ICMP Timestamp (Type 13) and Timestamp Reply (Type 14) are designed to measure clock synchronization and one-way network delays between hosts. Unlike ICMP Echo which only measures round-trip time, timestamps include originate, receive, and transmit time fields that allow computation of one-way delays. However, they also leak timing information and are typically disabled.

## ICMP Timestamp Packet Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type (13/14)  |   Code (0)    |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Identifier          |        Sequence Number        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Originate Timestamp (ms since midnight UTC) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Receive Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Transmit Timestamp                          |
```

## Sending an ICMP Timestamp Request

```bash
# Most modern systems don't have a dedicated timestamp tool
# Use hping3 to send ICMP timestamp requests
apt install hping3

# Send ICMP Timestamp Request
hping3 --icmp --icmp-type 13 -c 3 10.20.0.1

# Capture the exchange
tcpdump -i eth0 -n -v 'icmp[0]=13 or icmp[0]=14'
```

## Using Python to Send Timestamp Requests

```python
import socket
import struct
import time

def send_icmp_timestamp(dest_ip):
    """Send an ICMP Timestamp Request and print the reply."""
    # Create raw socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
    sock.settimeout(2.0)

    # Current time in ms since midnight UTC
    now = int((time.time() % 86400) * 1000)

    # Build ICMP Timestamp Request (type=13, code=0)
    # checksum=0 initially, will be filled
    header = struct.pack('!BBHHH', 13, 0, 0, 1, 1)
    timestamps = struct.pack('!III', now, 0, 0)  # originate, receive, transmit

    packet = header + timestamps
    # Calculate checksum
    if len(packet) % 2:
        packet += b'\x00'
    s = sum(struct.unpack('!%dH' % (len(packet) // 2), packet))
    s = (s >> 16) + (s & 0xffff)
    s += s >> 16
    checksum = ~s & 0xffff

    packet = struct.pack('!BBHHH', 13, 0, checksum, 1, 1) + timestamps
    sock.sendto(packet, (dest_ip, 0))

    try:
        data, addr = sock.recvfrom(1024)
        icmp_type = data[20]
        if icmp_type == 14:  # Timestamp Reply
            _, _, _, orig, recv, trans = struct.unpack('!HHIIII', data[20:40])
            print(f"Reply from {addr[0]}:")
            print(f"  Originate: {orig} ms")
            print(f"  Receive:   {recv} ms")
            print(f"  Transmit:  {trans} ms")
    except socket.timeout:
        print("No reply — timestamps likely disabled")

# Example: send_icmp_timestamp("192.168.1.1")
```

## Blocking ICMP Timestamps for Security

ICMP timestamps can be used to fingerprint OS clock offsets and assist in timing attacks:

```bash
# Block ICMP Timestamp Requests (Type 13)
iptables -A INPUT -p icmp --icmp-type timestamp-request -j DROP

# Block Timestamp Replies (Type 14) — prevent information leakage
iptables -A OUTPUT -p icmp --icmp-type timestamp-reply -j DROP

# Verify with nmap scanner check
nmap --script icmp-timestamp 10.20.0.1
# If blocked, nmap will report timestamps are filtered
```

## Conclusion

ICMP Timestamp messages are a legacy mechanism largely replaced by NTP for time synchronization. While they provide useful one-way delay measurement capability, the security trade-off (timing fingerprinting, clock inference) makes them worth disabling on public-facing interfaces. Block Types 13 and 14 in your firewall while keeping the operationally critical ICMP types (echo, time-exceeded, fragmentation-needed) open.
