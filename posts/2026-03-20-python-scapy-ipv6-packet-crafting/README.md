# How to Use Python scapy for IPv6 Packet Crafting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Scapy, IPv6, Packet Crafting, Network Testing, Security

Description: Use Python's scapy library to craft, send, and capture IPv6 packets for network testing, protocol analysis, and security research.

## Installing scapy

```bash
pip install scapy

# On Linux, scapy needs root or CAP_NET_RAW for raw socket access

# Run scripts with sudo or grant capabilities
```

## Basic IPv6 Packet Crafting

```python
from scapy.all import *
from scapy.layers.inet6 import IPv6, ICMPv6EchoRequest, ICMPv6EchoReply

# Create a basic IPv6 packet
packet = IPv6(
    src="2001:db8::1",    # Source address
    dst="2001:db8::2",    # Destination address
    hlim=64               # Hop limit (TTL equivalent)
)

# View packet fields
print(packet.show())
print(f"Packet bytes: {bytes(packet).hex()}")
```

## ICMPv6 Echo (Ping)

```python
from scapy.all import *
from scapy.layers.inet6 import IPv6, ICMPv6EchoRequest

def ping6_scapy(destination: str, count: int = 3):
    """Send ICMPv6 echo requests and display responses."""
    for i in range(count):
        # Build the packet: IPv6 header + ICMPv6 Echo Request
        packet = IPv6(dst=destination) / ICMPv6EchoRequest(id=i, seq=i)

        # sr1 = send and receive 1 response (timeout=2 seconds)
        response = sr1(packet, timeout=2, verbose=False)

        if response:
            print(f"Reply from {response[IPv6].src}: seq={i}")
        else:
            print(f"Request timeout for seq={i}")

# ping6_scapy("2001:4860:4860::8888")
```

## NDP Neighbor Solicitation

```python
from scapy.all import *
from scapy.layers.inet6 import (
    IPv6, ICMPv6ND_NS, ICMPv6ND_NA,
    ICMPv6NDOptSrcLLAddr
)

def send_neighbor_solicitation(target_ip: str, interface: str):
    """
    Send an NDP Neighbor Solicitation to discover the MAC address
    of a target IPv6 address.
    """
    # Solicited-node multicast: ff02::1:ff<last 24 bits of target>
    target_bytes = bytes(ipaddress.IPv6Address(target_ip).packed)
    solicited_multicast = "ff02::1:ff" + ":".join(
        f"{target_bytes[-3]:02x}{target_bytes[-2]:02x}:{target_bytes[-1]:02x}00"
        .split(":")
    )

    packet = (
        Ether(dst="33:33:ff:xx:xx:xx") /  # Multicast MAC
        IPv6(dst=solicited_multicast) /
        ICMPv6ND_NS(tgt=target_ip) /
        ICMPv6NDOptSrcLLAddr(lladdr=get_if_hwaddr(interface))
    )

    # Send and wait for Neighbor Advertisement
    response = srp1(packet, iface=interface, timeout=2, verbose=False)
    if response:
        print(f"Target {target_ip} is at {response[Ether].src}")
    else:
        print(f"No response from {target_ip}")
```

## TCP SYN Scan over IPv6

```python
from scapy.all import *
from scapy.layers.inet6 import IPv6

def tcp_syn_scan_ipv6(target: str, ports: list[int]) -> dict[int, str]:
    """
    Perform a TCP SYN scan against IPv6 target.
    Returns dict of {port: state} where state is 'open', 'closed', or 'filtered'
    """
    results = {}

    for port in ports:
        packet = IPv6(dst=target) / TCP(dport=port, flags="S")
        response = sr1(packet, timeout=2, verbose=False)

        if response is None:
            results[port] = "filtered"
        elif response.haslayer(TCP):
            if response[TCP].flags & 0x12:  # SYN-ACK
                results[port] = "open"
                # Send RST to clean up the connection
                sr1(IPv6(dst=target) / TCP(dport=port, flags="R"),
                    timeout=1, verbose=False)
            elif response[TCP].flags & 0x04:  # RST
                results[port] = "closed"
        else:
            results[port] = "unknown"

    return results
```

## Sniffing IPv6 Traffic

```python
from scapy.all import sniff
from scapy.layers.inet6 import IPv6, ICMPv6ND_NS

def capture_ipv6_ndp(interface: str, duration: int = 10):
    """Capture and display NDP Neighbor Solicitation messages."""
    print(f"Capturing NDP traffic on {interface} for {duration}s...")

    def process_packet(pkt):
        if pkt.haslayer(IPv6):
            src = pkt[IPv6].src
            dst = pkt[IPv6].dst
            if pkt.haslayer(ICMPv6ND_NS):
                target = pkt[ICMPv6ND_NS].tgt
                print(f"NDP NS: {src} looking for {target}")

    # Filter: only IPv6 packets with ICMPv6
    sniff(
        iface=interface,
        filter="ip6",
        prn=process_packet,
        timeout=duration
    )
```

## Crafting IPv6 Extension Headers

```python
from scapy.all import *
from scapy.layers.inet6 import IPv6, IPv6ExtHdrHopByHop, ICMPv6EchoRequest

# Create packet with Hop-by-Hop extension header
packet = (
    IPv6(dst="2001:db8::1") /
    IPv6ExtHdrHopByHop(options=[]) /  # Empty HBH extension header
    ICMPv6EchoRequest()
)

print(packet.show())
```

## Conclusion

Scapy makes IPv6 packet crafting accessible in Python. From ICMPv6 pings to TCP SYN scans and NDP message construction, scapy handles IPv6 extension headers, checksum computation, and Layer 2 framing automatically. Use it for protocol testing, network security research, and custom network tool development - always in authorized environments.
