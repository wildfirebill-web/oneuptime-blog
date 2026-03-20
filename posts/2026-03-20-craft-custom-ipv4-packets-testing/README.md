# How to Craft Custom IPv4 Packets for Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Packet Crafting, Security Testing, Scapy, Network Testing

Description: Crafting custom IPv4 packets enables engineers to test firewall rules, verify routing behavior, reproduce edge cases, and validate network device responses to malformed or unusual traffic.

## Use Cases for Custom Packets

- Testing firewall ACLs and IDS/IPS rules
- Simulating specific TTL values for traceroute tools
- Reproducing fragmentation behavior
- Validating QoS markings traverse the network correctly
- Stress-testing packet reassembly buffers

## Tool Options

| Tool | Language | Best For |
|------|----------|----------|
| Scapy | Python | Full packet control |
| hping3 | CLI | Fast TCP/ICMP probing |
| nping | CLI (Nmap) | Flood testing, timing |
| iperf3 | CLI | Bandwidth and MTU testing |
| nc (netcat) | CLI | Application payload testing |

## Custom Packet with Specific DSCP and TTL

```python
from scapy.all import IP, UDP, Raw, send

# Test that DSCP EF marking survives across your network

EF_DSCP = 46
tos_byte = EF_DSCP << 2  # 0xB8

pkt = IP(
    dst="10.20.30.1",
    ttl=30,
    tos=tos_byte,           # DSCP EF
    flags="DF",             # Don't fragment
    id=0xABCD,              # Recognizable ID for filtering
) / UDP(sport=5004, dport=5004) / Raw(b'\xAB' * 160)

send(pkt, verbose=False)
print("Sent custom DSCP EF test packet")
```

## Testing Firewall Rules with hping3

```bash
# Send TCP SYN to port 443 with spoofed source IP
sudo hping3 -S -p 443 --spoof 192.0.2.1 10.0.0.1

# ICMP flood test (use carefully, only on test networks)
sudo hping3 --icmp --flood --count 1000 10.0.0.1

# Send UDP packet with specific TTL
sudo hping3 --udp -p 53 --ttl 10 8.8.8.8
```

## Crafting an ICMP Packet with Custom Data

```python
from scapy.all import IP, ICMP, Raw, sr1

# Send ICMP echo with identifiable payload
payload = b"TESTPACKET-001"
pkt = IP(dst="192.168.1.1") / ICMP(id=0x1234, seq=1) / Raw(payload)
reply = sr1(pkt, timeout=2, verbose=False)

if reply:
    echo_data = bytes(reply[Raw])
    print(f"Echo reply payload: {echo_data}")
    assert echo_data == payload, "Payload was modified in transit!"
```

## Testing Fragmentation Handling

```python
from scapy.all import IP, ICMP, Raw, sr1

# Send a large ICMP packet with DF set - should return ICMP Fragmentation Needed
oversized = IP(dst="192.168.1.1", flags="DF") / ICMP() / Raw(b"X" * 2000)
reply = sr1(oversized, timeout=2, verbose=False)

if reply and reply[ICMP].type == 3 and reply[ICMP].code == 4:
    mtu = reply[ICMP].nexthopmtu
    print(f"Fragmentation Needed! Suggested MTU: {mtu}")
```

## Creating Malformed Packets for IDS Testing

```python
from scapy.all import IP, TCP, send

# Packet with invalid IHL (claims 60-byte header but only sends 20)
malformed = IP(ihl=15, dst="10.0.0.1") / TCP(dport=80, flags="S")
send(malformed, verbose=False)
print("Sent malformed IHL packet - check IDS alerts")
```

## Key Takeaways

- Custom packets enable precise testing of firewall rules, QoS markings, and edge cases.
- Scapy gives full Python-level control; `hping3` provides fast CLI-based probing.
- Always test on isolated or authorized networks - packet crafting can trigger IDS alerts.
- Use recognizable ID values or payloads to easily identify test packets in captures.
