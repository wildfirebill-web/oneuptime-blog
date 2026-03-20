# How to Understand the ECN Bits in the IPv4 Header

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, ECN, Networking, Congestion Control, TCP/IP, QoS

Description: The two ECN bits in the IPv4 ToS byte allow routers to signal congestion without dropping packets, enabling endpoints to reduce transmission rates and improve network efficiency.

## What Is ECN?

Explicit Congestion Notification (ECN), defined in RFC 3168, uses the last 2 bits of the IPv4 ToS byte (bits 1-0) for congestion signaling:

| ECN Codepoint | Value | Meaning |
|---------------|-------|---------|
| Not-ECT | 00 | ECN not supported by the transport |
| ECT(1) | 01 | ECN-capable transport (alternative) |
| ECT(0) | 10 | ECN-capable transport (preferred) |
| CE | 11 | Congestion Experienced |

## ECN Flow

```mermaid
sequenceDiagram
    participant Sender
    participant Router
    participant Receiver

    Sender->>Router: Packet (ECT=10)
    Note over Router: Buffer filling up
    Router->>Receiver: Packet (CE=11, not dropped)
    Receiver->>Sender: TCP ACK with ECE flag set
    Sender->>Sender: Reduce congestion window (like a loss)
    Sender->>Receiver: TCP packet with CWR flag set
```

## Enabling ECN on Linux

```bash
# Check current ECN setting (0=off, 1=on when both sides support, 2=always)
cat /proc/sys/net/ipv4/tcp_ecn

# Enable ECN for connections where both sides agree
sudo sysctl -w net.ipv4.tcp_ecn=1

# Force ECN even when the remote doesn't request it (use carefully)
sudo sysctl -w net.ipv4.tcp_ecn=2

# Persist the setting
echo "net.ipv4.tcp_ecn = 1" | sudo tee -a /etc/sysctl.conf
```

## Checking ECN in Python

```python
import socket

# Check if a socket has ECN enabled
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("example.com", 80))

# Read the current IP_TOS setting
tos = sock.getsockopt(socket.IPPROTO_IP, socket.IP_TOS)
ecn = tos & 0x03  # Bottom 2 bits
print(f"ToS=0x{tos:02X}  ECN bits={ecn:02b}")
sock.close()
```

## Reading ECN from Captured Packets

```bash
# Show ECN bits with tshark
tshark -i eth0 -T fields -e ip.dsfield.ecn -Y "ip"

# Filter for CE-marked packets (congestion experienced)
tshark -r capture.pcap -Y "ip.dsfield.ecn == 3"

# Display ECN with tcpdump verbose output
tcpdump -i eth0 -v -n tcp | grep ECN
```

## ECN and UDP

ECN is transport-agnostic at the IP level. For UDP-based transports (QUIC, WebRTC, DTLS), the application must handle ECN signaling explicitly using `IP_RECVTOS` / `IP_TOS` socket options.

```bash
# Verify ECN marking on QUIC traffic
tcpdump -i eth0 -v 'udp port 443' | grep 'tos 0x'
```

## Key Takeaways

- ECN uses 2 bits of the IPv4 ToS byte to signal congestion without packet drops.
- CE (11) is set by a congested router; endpoints use TCP ECE/CWR flags to act on it.
- Enable ECN on Linux with `net.ipv4.tcp_ecn=1` for mutual agreement, or `=2` to always request it.
- ECN reduces retransmission-induced latency spikes while maintaining throughput.
