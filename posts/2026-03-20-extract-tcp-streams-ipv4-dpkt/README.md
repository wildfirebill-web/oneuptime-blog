# How to Extract TCP Streams from IPv4 Traffic with dpkt

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dpkt, TCP, IPv4, Packet Analysis, Python, Network Forensics, PCAP

Description: Use the Python dpkt library to parse PCAP files, reassemble TCP streams from IPv4 traffic, and extract application-layer data from raw packet captures.

## Introduction

`dpkt` is a lightweight Python library for packet parsing and creation. Unlike Scapy (which is feature-rich but heavy), dpkt is fast and memory-efficient - ideal for processing large PCAP files. This guide shows how to extract and reassemble TCP streams from IPv4 captures.

## Prerequisites

```bash
pip install dpkt
```

## Reading a PCAP File

```python
import dpkt
import socket
from collections import defaultdict

def ip_to_str(addr):
    """Convert binary IP address to dotted-decimal string."""
    return socket.inet_ntoa(addr)

# Open and iterate over packets in a PCAP file

with open("capture.pcap", "rb") as f:
    pcap = dpkt.pcap.Reader(f)
    
    for ts, buf in pcap:
        try:
            # Parse Ethernet frame
            eth = dpkt.ethernet.Ethernet(buf)
            
            # Check for IPv4
            if not isinstance(eth.data, dpkt.ip.IP):
                continue
            
            ip = eth.data
            src = ip_to_str(ip.src)
            dst = ip_to_str(ip.dst)
            
            print(f"{ts:.3f} {src} -> {dst} proto={ip.p} len={ip.len}")
        
        except Exception as e:
            continue
```

## Extracting TCP Streams

A TCP stream is identified by its 4-tuple (src_ip, src_port, dst_ip, dst_port). To reassemble, we collect payload bytes per stream ordered by sequence number:

```python
import dpkt, socket
from collections import defaultdict

def extract_tcp_streams(pcap_file):
    """Reassemble TCP streams from a pcap file."""
    # streams[4-tuple] = {seq: data}
    streams = defaultdict(dict)
    
    with open(pcap_file, "rb") as f:
        pcap = dpkt.pcap.Reader(f)
        
        for ts, buf in pcap:
            try:
                eth = dpkt.ethernet.Ethernet(buf)
                if not isinstance(eth.data, dpkt.ip.IP):
                    continue
                
                ip = eth.data
                if not isinstance(ip.data, dpkt.tcp.TCP):
                    continue
                
                tcp = ip.data
                
                # Skip packets without payload
                if len(tcp.data) == 0:
                    continue
                
                src = socket.inet_ntoa(ip.src)
                dst = socket.inet_ntoa(ip.dst)
                
                # Normalize stream key (client -> server direction)
                if tcp.dport < tcp.sport:
                    flow_key = (dst, tcp.dport, src, tcp.sport)
                else:
                    flow_key = (src, tcp.sport, dst, tcp.dport)
                
                # Store payload indexed by sequence number
                streams[flow_key][tcp.seq] = tcp.data
            
            except Exception:
                continue
    
    return streams

# Reassemble each stream in sequence order
streams = extract_tcp_streams("capture.pcap")

for flow_key, segments in streams.items():
    src_ip, src_port, dst_ip, dst_port = flow_key
    
    # Sort by sequence number and concatenate
    sorted_data = b''.join(data for _, data in sorted(segments.items()))
    
    print(f"\n=== Stream: {src_ip}:{src_port} -> {dst_ip}:{dst_port} ===")
    print(f"Total bytes: {len(sorted_data)}")
    
    # Try to decode as text
    try:
        text = sorted_data.decode('utf-8', errors='replace')
        print(text[:200])
    except Exception:
        print(f"Binary data (hex preview): {sorted_data[:32].hex()}")
```

## Filtering HTTP Streams

```python
import dpkt, socket
from collections import defaultdict

def find_http_streams(pcap_file):
    """Find all HTTP (port 80) streams in a pcap."""
    http_streams = {}
    
    with open(pcap_file, "rb") as f:
        pcap = dpkt.pcap.Reader(f)
        flows = defaultdict(bytearray)
        
        for ts, buf in pcap:
            try:
                eth = dpkt.ethernet.Ethernet(buf)
                ip = eth.data
                if not isinstance(ip, dpkt.ip.IP):
                    continue
                tcp = ip.data
                if not isinstance(tcp, dpkt.tcp.TCP):
                    continue
                
                # Only interested in port 80 traffic
                if tcp.dport != 80 and tcp.sport != 80:
                    continue
                
                if len(tcp.data) == 0:
                    continue
                
                src = socket.inet_ntoa(ip.src)
                dst = socket.inet_ntoa(ip.dst)
                key = f"{src}:{tcp.sport}->{dst}:{tcp.dport}"
                
                flows[key] += tcp.data
                
            except Exception:
                continue
    
    return dict(flows)

http_flows = find_http_streams("capture.pcap")
for key, data in http_flows.items():
    text = data.decode('utf-8', errors='replace')
    if 'HTTP' in text:
        print(f"\n--- {key} ---")
        print(text[:300])
```

## Writing Extracted Streams to Files

```python
import os

output_dir = "extracted_streams"
os.makedirs(output_dir, exist_ok=True)

for i, (key, data) in enumerate(http_flows.items()):
    filename = f"{output_dir}/stream_{i:04d}.txt"
    with open(filename, "wb") as f:
        f.write(data)
    print(f"Saved: {filename} ({len(data)} bytes)")
```

## Conclusion

`dpkt` provides efficient, low-level access to PCAP data for TCP stream extraction. Its minimal dependencies and fast parsing make it ideal for processing large captures in forensic analysis, security research, or protocol debugging workflows.
