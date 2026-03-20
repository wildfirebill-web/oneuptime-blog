# How to Capture and Analyze HTTP Traffic over IPv4 Using Scapy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Scapy, HTTP, IPv4, Packet Analysis, Security, Python, Network Monitoring

Description: Use Scapy to capture live HTTP traffic over IPv4, extract request/response data, and analyze headers and payloads for security auditing or debugging.

## Introduction

Scapy can capture and decode HTTP traffic at the packet level, giving you visibility into cleartext HTTP sessions. This is useful for security auditing, debugging, API testing, and understanding application behavior on the network.

> **Note**: Only capture traffic on networks you own or have authorization to monitor. Intercepting traffic without consent is illegal.

## Prerequisites

```bash
pip install scapy
# Root/sudo required for raw socket capture

```

## Capturing HTTP Request Packets

```python
from scapy.all import sniff, IP, TCP, Raw

def analyze_http(pkt):
    """Extract and display HTTP requests from captured packets."""
    if not (pkt.haslayer(IP) and pkt.haslayer(TCP) and pkt.haslayer(Raw)):
        return
    
    payload = pkt[Raw].load
    
    # Identify HTTP requests (methods are ASCII at the start)
    try:
        payload_str = payload.decode('utf-8', errors='replace')
    except Exception:
        return
    
    http_methods = ('GET ', 'POST ', 'PUT ', 'DELETE ', 'HEAD ', 'OPTIONS ', 'PATCH ')
    
    if any(payload_str.startswith(m) for m in http_methods):
        src_ip = pkt[IP].src
        dst_ip = pkt[IP].dst
        dst_port = pkt[TCP].dport
        
        # Extract the first line of the request
        first_line = payload_str.split('\r\n')[0]
        
        # Extract the Host header
        host = ''
        for line in payload_str.split('\r\n'):
            if line.lower().startswith('host:'):
                host = line.split(':', 1)[1].strip()
                break
        
        print(f"[HTTP REQUEST] {src_ip} -> {dst_ip}:{dst_port}")
        print(f"  Request: {first_line}")
        print(f"  Host: {host}")
        print()

# Capture HTTP traffic (port 80)
print("Capturing HTTP traffic on port 80... (Ctrl+C to stop)")
sniff(filter="tcp port 80 and ip", prn=analyze_http, store=False)
```

## Capturing HTTP Responses

```python
from scapy.all import sniff, IP, TCP, Raw

def analyze_http_response(pkt):
    """Extract HTTP response status codes from captured traffic."""
    if not (pkt.haslayer(IP) and pkt.haslayer(TCP) and pkt.haslayer(Raw)):
        return
    
    try:
        payload = pkt[Raw].load.decode('utf-8', errors='replace')
    except Exception:
        return
    
    if payload.startswith('HTTP/'):
        src_ip = pkt[IP].src
        dst_ip = pkt[IP].dst
        status_line = payload.split('\r\n')[0]
        
        # Extract Content-Type
        content_type = ''
        for line in payload.split('\r\n'):
            if line.lower().startswith('content-type:'):
                content_type = line.split(':', 1)[1].strip()
                break
        
        print(f"[HTTP RESPONSE] {src_ip} -> {dst_ip}")
        print(f"  Status: {status_line}")
        print(f"  Content-Type: {content_type}")
        print()

sniff(filter="tcp port 80 and ip", prn=analyze_http_response, store=False)
```

## Extracting Full HTTP Sessions from a PCAP File

Analyze previously captured traffic:

```python
from scapy.all import rdpcap, IP, TCP, Raw
from collections import defaultdict

# Load a pcap file
packets = rdpcap("capture.pcap")

# Group packets by TCP flow (4-tuple)
flows = defaultdict(list)
for pkt in packets:
    if pkt.haslayer(IP) and pkt.haslayer(TCP) and pkt.haslayer(Raw):
        src = (pkt[IP].src, pkt[TCP].sport)
        dst = (pkt[IP].dst, pkt[TCP].dport)
        flow_key = (src, dst) if pkt[TCP].dport in (80, 8080) else (dst, src)
        flows[flow_key].append(pkt[Raw].load)

# Reassemble and display HTTP conversations
for flow, payloads in flows.items():
    combined = b''.join(payloads)
    try:
        text = combined.decode('utf-8', errors='replace')
        if 'HTTP' in text:
            print(f"\n=== Flow: {flow[0][0]}:{flow[0][1]} -> {flow[1][0]}:{flow[1][1]} ===")
            print(text[:500])  # Print first 500 chars
    except Exception:
        pass
```

## Extracting URLs and Headers to a CSV

```python
import csv
from scapy.all import sniff, IP, TCP, Raw
from datetime import datetime

output_file = "http_log.csv"
fieldnames = ['timestamp', 'src_ip', 'dst_ip', 'method', 'host', 'path', 'user_agent']

with open(output_file, 'w', newline='') as csvfile:
    writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
    writer.writeheader()
    
    def log_http(pkt):
        if not (pkt.haslayer(TCP) and pkt.haslayer(Raw)):
            return
        try:
            payload = pkt[Raw].load.decode('utf-8', errors='replace')
        except Exception:
            return
        
        if not any(payload.startswith(m) for m in ('GET ', 'POST ', 'PUT ')):
            return
        
        lines = payload.split('\r\n')
        req_parts = lines[0].split(' ')
        headers = {l.split(':', 1)[0].lower(): l.split(':', 1)[1].strip()
                   for l in lines[1:] if ':' in l}
        
        writer.writerow({
            'timestamp': datetime.now().isoformat(),
            'src_ip': pkt[IP].src,
            'dst_ip': pkt[IP].dst,
            'method': req_parts[0],
            'host': headers.get('host', ''),
            'path': req_parts[1] if len(req_parts) > 1 else '',
            'user_agent': headers.get('user-agent', '')
        })
        csvfile.flush()
    
    sniff(filter="tcp port 80 and ip", prn=log_http, store=False)
```

## Conclusion

Scapy provides a powerful Python interface for HTTP traffic analysis. For encrypted HTTPS traffic, you would need TLS key logging or a MITM proxy. Use these capabilities only for authorized security testing, debugging, and network monitoring.
