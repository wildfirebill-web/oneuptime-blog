# How to Spoof IPv4 Source Addresses with Scapy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Scapy, IPv4, Security Testing, Packet Crafting, Python, Networking

Description: Craft and send IPv4 packets with spoofed source addresses using Scapy for legitimate security testing, firewall rule validation, and network research purposes.

## Introduction

IP source address spoofing is the technique of sending packets with a forged source IP address. While malicious actors use it for DDoS amplification and reflection attacks, security professionals use it legitimately to test firewall rules, validate BCP38 enforcement, and research network protocols. This guide covers Scapy-based spoofing for authorized testing only.

> **Legal and Ethical Notice**: Only send spoofed packets on networks you own or have explicit written permission to test. Unauthorized spoofing is illegal in most jurisdictions.

## Prerequisites

```bash
pip install scapy
# Root/sudo required for raw socket access

```

## Sending a Single Spoofed ICMP Packet

```python
from scapy.all import IP, ICMP, send

# Craft a packet with a spoofed source IP
pkt = IP(
    src="192.0.2.1",      # Spoofed source IP (not your real IP)
    dst="10.0.0.1"        # Target IP
) / ICMP()

# Send the packet (requires root)
send(pkt, verbose=True)
```

## Spoofed TCP SYN Packet

Useful for testing firewall SYN filtering:

```python
from scapy.all import IP, TCP, send
import random

pkt = IP(
    src="198.51.100.50",   # Spoofed source
    dst="10.0.0.10"        # Target
) / TCP(
    sport=random.randint(1024, 65535),   # Random source port
    dport=80,                             # Target port
    flags="S"                             # SYN flag only
)

send(pkt)
```

## Testing BCP38 Enforcement (Anti-Spoofing Validation)

BCP38 (Network Ingress Filtering) requires ISPs and network operators to drop packets with source IPs that cannot originate from the customer's subnet. Test whether your router enforces it:

```python
from scapy.all import IP, ICMP, send

# Send a packet with a source IP from a completely different range
# Should be dropped if BCP38 is enforced on your router
pkt = IP(
    src="1.2.3.4",         # External IP - should not originate from your subnet
    dst="8.8.8.8"          # Send toward the internet
) / ICMP()

send(pkt, iface="eth0")
# If this packet leaves your network, BCP38 is NOT enforced
```

## Sending Multiple Spoofed Packets

```python
from scapy.all import IP, UDP, send, RandShort
import ipaddress

# Send UDP packets with rotating spoofed source IPs
network = ipaddress.ip_network("203.0.113.0/24")
hosts = list(network.hosts())

for src_ip in hosts[:10]:
    pkt = IP(src=str(src_ip), dst="10.0.0.1") / UDP(dport=53) / b"\x00" * 10
    send(pkt, verbose=False)
    print(f"Sent packet from {src_ip}")
```

## Validating Firewall Rules

Test that your firewall correctly drops spoofed traffic from untrusted ranges:

```python
from scapy.all import IP, TCP, sr1

# Send a SYN from a private range - should be blocked by the firewall
probe = IP(src="192.168.99.1", dst="TARGET_IP") / TCP(dport=443, flags="S")
response = sr1(probe, timeout=2, verbose=False)

if response is None:
    print("No response - firewall is blocking (expected)")
else:
    print(f"Got response: {response.summary()} - check your firewall rules")
```

## Capturing Responses to Spoofed Packets

For testing scenarios where you want to see if responses come back to the spoofed IP (reflection), capture on a separate interface:

```python
from scapy.all import sniff, IP, ICMP, send
import threading

def send_spoofed():
    pkt = IP(src="10.0.0.99", dst="10.0.0.1") / ICMP()
    send(pkt)

# Start a sniffer looking for responses to the spoofed IP
def capture():
    pkts = sniff(filter="host 10.0.0.99 and icmp", count=5, timeout=3)
    for p in pkts:
        print(f"Response: {p.summary()}")

t = threading.Thread(target=capture)
t.start()
send_spoofed()
t.join()
```

## Conclusion

Scapy makes IP spoofing trivially simple, which highlights why network operators must enforce BCP38 ingress filtering. Always use these techniques only in authorized test environments, and ensure your network infrastructure drops spoofed packets at the border to protect against abuse.
