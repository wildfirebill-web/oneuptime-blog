# How to Build a Network Scanner for IPv4 Hosts Using Scapy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Scapy, Network Scanner, IPv4, ARP, ICMP, Security

Description: Learn how to build a network scanner that discovers live IPv4 hosts using ARP and ICMP techniques with Scapy in Python.

## Network Discovery Methods

Two primary approaches for discovering live hosts:
1. **ARP Scan** — works only on the local subnet; fast and reliable
2. **ICMP Ping Sweep** — works across routed networks; may be blocked by firewalls

## Method 1: ARP Host Discovery (Local Subnet)

ARP scanning is the most reliable method on local networks:

```python
from scapy.all import ARP, Ether, srp
import ipaddress

def arp_scan(network: str) -> list[dict]:
    """
    Discover live hosts on a local subnet using ARP.
    network: e.g., "192.168.1.0/24"
    """
    # Ether broadcast + ARP who-has for all IPs in the subnet
    arp_request = ARP(pdst=network)
    broadcast = Ether(dst="ff:ff:ff:ff:ff:ff")
    packet = broadcast / arp_request

    # srp() operates at Layer 2; timeout=3 seconds
    answered, _ = srp(packet, timeout=3, verbose=False)

    hosts = []
    for sent, received in answered:
        hosts.append({
            "ip": received[ARP].psrc,
            "mac": received[ARP].hwsrc,
        })

    return sorted(hosts, key=lambda h: ipaddress.IPv4Address(h["ip"]))


results = arp_scan("192.168.1.0/24")
print(f"Discovered {len(results)} hosts:\n")
print(f"{'IP':<18} {'MAC'}")
print("-" * 35)
for host in results:
    print(f"{host['ip']:<18} {host['mac']}")
```

## Method 2: ICMP Ping Sweep

For scanning beyond the local subnet:

```python
from scapy.all import IP, ICMP, sr
import ipaddress

def icmp_sweep(network: str, timeout: int = 2) -> list[str]:
    """Discover live hosts using ICMP ping sweep."""
    net = ipaddress.IPv4Network(network, strict=False)
    # Skip network and broadcast addresses
    targets = [str(ip) for ip in net.hosts()]

    # Build all ICMP packets at once
    packets = [IP(dst=ip) / ICMP() for ip in targets]

    # Send all packets and collect replies
    answered, _ = sr(packets, timeout=timeout, verbose=False)

    alive = []
    for sent, received in answered:
        if ICMP in received and received[ICMP].type == 0:
            alive.append(received[IP].src)

    return sorted(alive, key=lambda ip: ipaddress.IPv4Address(ip))


alive = icmp_sweep("10.0.0.0/24")
print(f"Live hosts: {alive}")
```

## Combined Scanner with Service Detection

```python
from scapy.all import ARP, Ether, srp, IP, TCP, sr
import socket

def scan_with_ports(network: str, ports: list[int]) -> list[dict]:
    """ARP scan followed by port check on discovered hosts."""
    # Phase 1: ARP discovery
    arp_req = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=network)
    answered, _ = srp(arp_req, timeout=3, verbose=False)
    hosts = [{"ip": r[1][ARP].psrc, "mac": r[1][ARP].hwsrc, "ports": []} for r in answered]

    # Phase 2: TCP SYN scan on each discovered host
    for host in hosts:
        pkts = [IP(dst=host["ip"]) / TCP(dport=p, flags="S") for p in ports]
        ans, _ = sr(pkts, timeout=1, verbose=False)

        for sent, recv in ans:
            if TCP in recv and recv[TCP].flags == 0x12:  # SYN-ACK = open
                host["ports"].append(sent[TCP].dport)
                # Send RST to close gracefully
                from scapy.all import send
                send(IP(dst=host["ip"]) / TCP(dport=sent[TCP].dport, flags="R"), verbose=False)

    return hosts


results = scan_with_ports("192.168.1.0/24", [22, 80, 443, 3306, 8080])
for h in results:
    if h["ports"]:
        print(f"{h['ip']} ({h['mac']}) - Open ports: {h['ports']}")
```

## Conclusion

Scapy-based network scanning is fast and flexible. ARP scans are the most reliable for local subnets; ICMP sweeps work across routers but may be filtered. Always obtain authorization before scanning networks you don't own. For production use, consider `nmap` which is more optimized and handles edge cases better.
