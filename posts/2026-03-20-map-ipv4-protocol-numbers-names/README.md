# How to Map IPv4 Protocol Numbers to Protocol Names

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Protocol Numbers, TCP/IP, Packet Analysis

Description: IPv4 protocol numbers are 8-bit values in the IP header that identify the encapsulated upper-layer protocol, and mapping them to names is essential for packet analysis and firewall rule writing.

## What Are Protocol Numbers?

The Protocol field at byte 9 of the IPv4 header is an 8-bit integer. IANA maintains the authoritative registry at iana.org/assignments/protocol-numbers. The system file `/etc/protocols` on Linux/macOS provides a local lookup table.

## Key Protocol Numbers

| Number | Keyword | Protocol |
|--------|---------|----------|
| 0 | HOPOPT | IPv6 Hop-by-Hop Options |
| 1 | ICMP | Internet Control Message Protocol |
| 2 | IGMP | Internet Group Management Protocol |
| 4 | IP | IP in IP (encapsulation) |
| 6 | TCP | Transmission Control Protocol |
| 8 | EGP | Exterior Gateway Protocol |
| 17 | UDP | User Datagram Protocol |
| 27 | RDP | Reliable Data Protocol |
| 41 | IPv6 | IPv6 encapsulated in IPv4 |
| 43 | IPv6-Route | Routing Header for IPv6 |
| 47 | GRE | Generic Routing Encapsulation |
| 50 | ESP | Encapsulating Security Payload |
| 51 | AH | Authentication Header |
| 58 | IPv6-ICMP | ICMP for IPv6 |
| 89 | OSPF | Open Shortest Path First |
| 112 | VRRP | Virtual Router Redundancy Protocol |
| 132 | SCTP | Stream Control Transmission Protocol |

## Looking Up Protocol Names on Linux

```bash
# View the full protocols database
cat /etc/protocols

# Look up by number (e.g., 89 = OSPF)
grep "	89	" /etc/protocols

# Look up by name
grep "^ospf" /etc/protocols
```

## Python: Building a Protocol Lookup Table

```python
def load_protocols(path: str = "/etc/protocols") -> dict:
    """
    Load the protocols database into a dict: {number: name}.
    Ignores comment lines and blank lines.
    """
    proto_map = {}
    with open(path) as f:
        for line in f:
            line = line.split('#')[0].strip()
            if not line:
                continue
            parts = line.split()
            if len(parts) >= 2:
                name = parts[0]
                try:
                    number = int(parts[1])
                    proto_map[number] = name
                except ValueError:
                    continue
    return proto_map

protocols = load_protocols()

# Look up protocol numbers from a list
for num in [1, 6, 17, 47, 89]:
    print(f"{num:3d} -> {protocols.get(num, 'unknown')}")
```

## Using socket.getprotobynumber()

Python's socket module provides a built-in lookup:

```python
import socket

for num in [1, 6, 17, 50, 89]:
    try:
        name = socket.getprotobynumber(num)
        print(f"{num} -> {name}")
    except OSError:
        print(f"{num} -> unknown")
```

## Filtering by Protocol in iptables

```bash
# Allow OSPF (89) between routers
iptables -A INPUT -p ospf -j ACCEPT

# Block GRE tunnels (47)
iptables -A INPUT -p gre -j DROP

# Allow ESP for IPsec
iptables -A INPUT -p esp -j ACCEPT
```

## Key Takeaways

- IPv4 protocol numbers are IANA-assigned 8-bit values identifying the encapsulated protocol.
- `/etc/protocols` provides a local human-readable mapping on Unix systems.
- Use `socket.getprotobynumber()` for Python lookups or parse `/etc/protocols` directly.
- Firewall rules can reference protocol names (e.g., `ospf`, `gre`) instead of numbers for readability.
