# How to Compare Scapy, dpkt, and PyShark for IPv4 Packet Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Scapy, dpkt, PyShark, IPv4, Packet Analysis, Networking

Description: A side-by-side comparison of Scapy, dpkt, and PyShark for IPv4 packet analysis, covering their strengths, weaknesses, and ideal use cases.

## Overview

Three libraries dominate Python IPv4 packet analysis:

- **Scapy**: All-in-one library for crafting, sending, sniffing, and dissecting packets
- **dpkt**: Lightweight, fast parser focused on reading and writing packets
- **PyShark**: Python wrapper around TShark (Wireshark CLI) with rich protocol dissection

## Feature Comparison

| Feature | Scapy | dpkt | PyShark |
|---------|-------|------|---------|
| Packet crafting | Excellent | Limited | None |
| Packet sending | Yes (root required) | No | No |
| Live capture | Yes | Via pcap lib | Yes |
| PCAP reading | Yes | Yes | Yes |
| Protocol coverage | ~300 | ~50 | ~3000 |
| Performance | Medium | Fast | Slowest |
| Dependencies | scapy | dpkt | TShark binary |
| Display filters | BPF only | BPF only | Full Wireshark syntax |

## Reading an IPv4 Packet from PCAP: Side by Side

### Scapy

```python
from scapy.all import rdpcap, IP, TCP

packets = rdpcap("capture.pcap")
for pkt in packets:
    if IP in pkt and TCP in pkt:
        print(f"{pkt[IP].src}:{pkt[TCP].sport} -> {pkt[IP].dst}:{pkt[TCP].dport}")
```

### dpkt

```python
import dpkt, socket

with open("capture.pcap", "rb") as f:
    for ts, raw in dpkt.pcap.Reader(f):
        try:
            eth = dpkt.ethernet.Ethernet(raw)
            if isinstance(eth.data, dpkt.ip.IP):
                ip = eth.data
                if isinstance(ip.data, dpkt.tcp.TCP):
                    tcp = ip.data
                    src = socket.inet_ntoa(ip.src)
                    dst = socket.inet_ntoa(ip.dst)
                    print(f"{src}:{tcp.sport} -> {dst}:{tcp.dport}")
        except Exception:
            pass
```

### PyShark

```python
import pyshark

cap = pyshark.FileCapture("capture.pcap", display_filter="tcp")
for pkt in cap:
    if hasattr(pkt, "ip") and hasattr(pkt, "tcp"):
        print(f"{pkt.ip.src}:{pkt.tcp.srcport} -> {pkt.ip.dst}:{pkt.tcp.dstport}")
cap.close()
```

## Performance Benchmark (Approximate)

Processing 1 million packets from PCAP:

```mermaid
bar
    title Packets/Second (higher is better)
    x-axis [dpkt, Scapy, PyShark]
    y-axis 0 --> 300000
    bar [250000, 50000, 5000]
```

- **dpkt**: ~250,000 pkt/s — fastest, minimal overhead
- **Scapy**: ~50,000 pkt/s — feature-rich, moderate overhead
- **PyShark**: ~5,000 pkt/s — subprocess-based, richest dissection

## When to Use Each

### Use Scapy when:
- You need to craft and send custom packets
- Building network tools (scanners, fuzzers, probes)
- Interactive exploration in a Python REPL
- Need protocol layering (`IP() / TCP() / Raw()`)

### Use dpkt when:
- Processing large PCAP files for statistics
- Speed is critical (log analysis pipelines)
- You only need basic protocol fields
- Minimal dependencies required

### Use PyShark when:
- You need Wireshark-quality protocol dissection
- Analyzing obscure protocols (VoIP, ICS, etc.)
- Using Wireshark display filter syntax
- Wireshark is already installed on the system

## Crafting a Packet (Scapy Only)

```python
from scapy.all import IP, TCP, send

# Only Scapy supports packet crafting
pkt = IP(dst="10.0.0.1") / TCP(dport=80, flags="S")
send(pkt)
```

## Conclusion

Choose your library based on the task: dpkt for high-speed PCAP processing, Scapy for packet crafting and research, and PyShark for rich protocol analysis leveraging Wireshark's dissectors. In many workflows, multiple libraries complement each other—use dpkt for bulk analysis and Scapy for crafting specific test packets.
