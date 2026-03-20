# How to Understand the Relationship Between MTU and MSS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, MSS, TCP, Networking, IPv4, Fragmentation

Description: Understand how MTU and TCP Maximum Segment Size (MSS) relate, how MSS is negotiated during TCP handshake, and how to configure MSS clamping to prevent fragmentation.

## Introduction

MTU (Maximum Transmission Unit) is a Layer 3 concept: the maximum IP packet size an interface can transmit. MSS (Maximum Segment Size) is a TCP option that limits how much data TCP puts in a single segment. They are related by: `MSS = MTU - IP header (20 bytes) - TCP header (20 bytes)`. Understanding this relationship is essential for preventing TCP fragmentation and optimizing TCP performance through tunnels and VPNs.

## MTU to MSS Calculation

```
MTU and MSS relationship:

  Ethernet MTU:      1500 bytes  (standard)
  IP header:          -20 bytes  (minimum, without options)
  TCP header:         -20 bytes  (minimum, without options)
  ─────────────────────────────
  Default MSS:        1460 bytes

  With VXLAN overhead:
  Physical MTU:       1500 bytes
  VXLAN overhead:      -50 bytes
  Tunnel MTU:         1450 bytes
  IP + TCP headers:    -40 bytes
  MSS through VXLAN:  1410 bytes

  With WireGuard:
  Physical MTU:       1500 bytes
  WireGuard overhead:  -80 bytes
  Tunnel MTU:         1420 bytes
  IP + TCP headers:    -40 bytes
  MSS through WG:     1380 bytes

  Jumbo frames:
  Ethernet MTU:       9000 bytes
  IP + TCP headers:    -40 bytes
  MSS with jumbo:     8960 bytes
```

## How MSS is Negotiated

```
TCP three-way handshake with MSS:

  Client                            Server
    │──── SYN ──────────────────────►│
    │   TCP Options: MSS=1460        │
    │                                │
    │◄─── SYN-ACK ───────────────────│
    │   TCP Options: MSS=1452        │  (server has lower MTU tunnel)
    │                                │
    │──── ACK ──────────────────────►│
    │                                │
  Both sides use MSS = min(1460, 1452) = 1452

Key points:
  - Each side announces its OWN receive MSS (limited by local interface MTU)
  - The sender uses the REMOTE side's announced MSS
  - MSS in SYN = MTU of outgoing interface - 40 bytes
  - Routers can clamp MSS (TCPMSS iptables rule)
  - MSS does NOT account for IP options or TCP options in data segments
```

## Observe MSS Negotiation

```bash
# Capture TCP handshake and inspect MSS:
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0' -v

# Look for lines like:
# options [mss 1460, ...] in SYN packets

# Alternative with tshark:
tshark -i eth0 -f 'tcp[tcpflags] & 0x02 != 0' \
  -T fields -e tcp.options.mss_val -e ip.src -e ip.dst
# Shows MSS value for each SYN/SYN-ACK

# Check MSS for established connection:
ss -tin | grep mss
# Output: mss:1460 snd_wnd:...
```

## Configure MSS Clamping with iptables

```bash
# MSS clamping rewrites MSS in SYN packets to prevent oversized segments:

# Clamp to specific value (for VXLAN with 1450 MTU):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -o vxlan0 -j TCPMSS --set-mss 1410

# Automatic clamping (clamps to path MTU - 40):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Apply to both directions on tunnel interface:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -i vxlan0 -j TCPMSS --set-mss 1410
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -o vxlan0 -j TCPMSS --set-mss 1410

# Verify:
iptables -t mangle -L FORWARD -v -n | grep TCPMSS

# Test result - capture should show clamped MSS:
tcpdump -i vxlan0 -n 'tcp[tcpflags] & tcp-syn != 0' -v | grep mss
```

## Set MSS via Socket Options

```python
#!/usr/bin/env python3
# Python: set MSS on TCP socket

import socket
import struct

# Create TCP socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# TCP_MAXSEG sets the MSS:
MSS = 1410  # For VXLAN with 1450 MTU
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_MAXSEG, MSS)

# Connect:
s.connect(('10.20.0.5', 8080))

# Verify MSS was used (check with tcpdump):
# tcpdump should show options [mss 1410,...] in SYN packet
```

```c
/* C: set MSS on TCP socket */
int sock = socket(AF_INET, SOCK_STREAM, 0);
int mss = 1410;
setsockopt(sock, IPPROTO_TCP, TCP_MAXSEG, &mss, sizeof(mss));
```

## Common MSS Values Reference

```
Interface Type      | Physical MTU | Overhead | Tunnel MTU | Correct MSS
--------------------|--------------|----------|------------|------------
Standard Ethernet   |    1500      |    0     |    1500    |    1460
VXLAN overlay       |    1500      |   50     |    1450    |    1410
GRE tunnel          |    1500      |   24     |    1476    |    1436
GRE + IPsec         |    1500      |   82     |    1418    |    1378
WireGuard           |    1500      |   80     |    1420    |    1380
AWS VPN (IPsec)     |    1500      |  101     |    1399    |    1359
Azure P2S VPN       |    1500      |  150     |    1350    |    1310
Jumbo Ethernet      |    9000      |    0     |    9000    |    8960
VXLAN on Jumbo      |    9000      |   50     |    8950    |    8910
```

## Conclusion

MSS equals MTU minus 40 bytes (IP + TCP headers). During TCP handshake, each side announces its MSS based on the local outgoing interface MTU. The sender is limited to `min(local-MSS, remote-announced-MSS)`. For tunnel interfaces, set the tunnel MTU correctly and apply MSS clamping with iptables to ensure new connections negotiate the right MSS automatically. Without MSS clamping, connections through tunnels may work for small data but stall on large transfers as TCP sends segments that exceed the tunnel MTU.
