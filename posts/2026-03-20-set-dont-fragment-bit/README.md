# How to Set the Don't Fragment (DF) Bit in IPv4 Packets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, DF Bit, Fragmentation, MTU, PMTUD, Linux, Networking

Description: Control the Don't Fragment bit in IPv4 packets using socket options, ping, and iptables to test PMTUD, discover path MTU, and prevent fragmentation.

## Introduction

The Don't Fragment (DF) bit in the IPv4 header instructs routers not to fragment the packet. If a router needs to forward a packet to a link with a smaller MTU, and the DF bit is set, the router must drop the packet and send an ICMP "Fragmentation Needed" message back to the sender. This mechanism is the foundation of Path MTU Discovery (PMTUD) and is used by TCP to find the optimal segment size for a path.

## DF Bit with ping

```bash
# Set DF bit with ping (-M do flag):

ping -M do -s 1472 10.20.0.5
# -M do: set DF bit (Don't Fragment)
# -s 1472: payload size (1472 + 28 bytes overhead = 1500 total)

# If succeeds: path MTU >= 1500 bytes
# If fails: "Frag needed and DF set" or "Message too long"
#           path MTU < 1500 (fragmentation required but blocked)

# Find exact path MTU by binary search:
# Try 1472 → success
# Try 1473 → fails
# Path MTU = 1472 + 28 = 1500 bytes

# For different link types:
ping -M do -s 1448 vpn-endpoint  # For GRE/IPsec VPN (typical)
ping -M do -s 1400 10.20.0.5    # For very conservative test
```

## DF Bit in Application Code

```python
#!/usr/bin/env python3
# Set DF bit on UDP socket

import socket

# IP_MTU_DISCOVER values:
IP_PMTUDISC_DONT = 0    # Don't set DF bit
IP_PMTUDISC_WANT = 1    # Set DF when possible (kernel may fragment)
IP_PMTUDISC_DO = 2      # Always set DF bit (kernel never fragments)
IP_PMTUDISC_PROBE = 3   # Like DO but also probe path MTU

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Set DF bit on all packets:
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, IP_PMTUDISC_DO)

# Enable path MTU discovery (probe):
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, IP_PMTUDISC_PROBE)

# Connect to destination (required to use IP_MTU after sending):
sock.connect(('10.20.0.5', 5000))

# Send packet - will fail with EMSGSIZE if too large for path MTU:
try:
    sock.send(b'X' * 1472)  # 1472 byte payload
    print("Sent successfully")
except OSError as e:
    if e.errno == 90:  # EMSGSIZE
        print(f"Packet too large! Current path MTU:")
        mtu = sock.getsockopt(socket.IPPROTO_IP, socket.IP_MTU)
        print(f"  Path MTU: {mtu} bytes")
        print(f"  Max UDP payload: {mtu - 28} bytes")
```

## DF Bit in TCP

```bash
# TCP always sets the DF bit by default on Linux
# TCP uses PMTUD to find optimal MSS

# Verify TCP DF behavior:
tcpdump -i eth0 -n 'tcp and host 10.20.0.5' -v 2>/dev/null | head -5
# Look for: DF (in the IP flags field)
# Standard TCP output: Flags [S] ... length 0, (DF)

# TCP PMTUD: when router sends ICMP Fragmentation Needed:
# TCP reduces MSS to fit the path MTU
# Watch for sudden reduction in TCP segment size after connecting:
ss -tin state established dst 10.20.0.5 | grep mss
# After PMTUD: mss value decreases
```

## Set DF Bit with iptables

```bash
# Mark specific flows to not be fragmented:
iptables -t mangle -A POSTROUTING -p udp --dport 5000 \
  -j MARK --set-mark 1

# Or directly manipulate DF bit in iptables (requires MARK + routing):
# More commonly done with tc or ip route commands

# To CLEAR the DF bit (allow fragmentation when needed):
# This is done in routing policy for specific interfaces
```

## Test ICMP Fragmentation Needed Reception

```bash
# When DF bit is set and packet is too large:
# Router sends ICMP type 3 code 4 (Fragmentation Needed)
# The ICMP message includes the MTU of the bottleneck link

# Capture ICMP Fragmentation Needed:
tcpdump -i eth0 -n 'icmp and icmp[0] = 3 and icmp[1] = 4'

# What the message contains:
# ICMP type 3 (Destination Unreachable)
# ICMP code 4 (Fragmentation Needed)
# Next-Hop MTU: the MTU of the link that would have fragmented

# If you send DF packets and get no ICMP back:
# → MTU black hole (router dropping without ICMP)
# → Fix: use tracepath to find black hole, reduce MTU manually
```

## Conclusion

The DF bit prevents fragmentation and triggers PMTUD when set. For ping-based MTU testing, use `-M do` flag. For applications, set `IP_PMTUDISC_DO` via socket options to enable PMTUD. TCP automatically handles this - it sets DF and adjusts MSS when ICMP Fragmentation Needed messages arrive. The most common problem is when ICMP Fragmentation Needed messages are blocked by firewalls, creating MTU black holes where TCP connections hang or UDP data is silently dropped.
