# How to Determine Maximum IPv4 Packet Size

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, MTU, Fragmentation, TCP/IP, Performance

Description: The maximum IPv4 packet size is 65,535 bytes as defined by the 16-bit Total Length field, but practical limits are set by link MTUs, path MTU discovery, and application requirements.

## Theoretical Maximum

The IPv4 Total Length field is 16 bits, allowing values from 0 to 65,535. Since the minimum IP header is 20 bytes, the maximum payload is 65,515 bytes. This theoretical maximum is rarely used because:

1. Most links have an MTU of 1500 bytes (Ethernet).
2. Fragmentation reduces throughput and increases reassembly overhead.
3. Some networks drop oversized datagrams.

## Common MTU Values

| Link Type | MTU (bytes) |
|-----------|-------------|
| Ethernet | 1500 |
| Wi-Fi (802.11) | 2304 (but usually 1500 in practice) |
| PPPoE | 1492 |
| Loopback (Linux) | 65536 |
| Jumbo frames | 9000 |
| FDDI | 4352 |

## Finding the MTU of an Interface

```bash
# Linux: show MTU for all interfaces
ip link show

# Or with ifconfig
ifconfig eth0 | grep mtu

# macOS
networksetup -getMTU en0
```

## Discovering Path MTU

Path MTU is the smallest MTU along the entire route to a destination. Use PMTUD:

```bash
# Linux: ping with DF set and progressively larger payloads
# -M do sets DF, -s sets payload size (total packet = s + 28 bytes IP+ICMP overhead)
ping -M do -s 1472 8.8.8.8   # 1472 + 28 = 1500 (should succeed on Ethernet)
ping -M do -s 1473 8.8.8.8   # 1501 bytes total — should fail if MTU is 1500

# View the cached path MTU to a destination
ip route get 8.8.8.8 | grep -i mtu
```

## Checking Path MTU with Python

```python
from scapy.all import IP, ICMP, Raw, sr1

def find_path_mtu(dst: str, max_size: int = 1500, min_size: int = 576) -> int:
    """Binary search for path MTU to destination."""
    low, high = min_size, max_size
    while low < high:
        mid = (low + high + 1) // 2
        payload_size = mid - 28  # 20 IP + 8 ICMP
        pkt = IP(dst=dst, flags="DF") / ICMP() / Raw(b"X" * payload_size)
        reply = sr1(pkt, timeout=2, verbose=False)
        if reply and reply[ICMP].type in (0, 3):
            low = mid  # Packet made it through
        else:
            high = mid - 1  # Fragmentation needed or no reply
    return low

pmtu = find_path_mtu("8.8.8.8")
print(f"Path MTU to 8.8.8.8: {pmtu} bytes")
```

## Jumbo Frames

For high-throughput internal networks, jumbo frames (MTU=9000) reduce per-packet overhead:

```bash
# Enable jumbo frames on Linux (requires NIC and switch support)
sudo ip link set eth0 mtu 9000
```

## Key Takeaways

- The theoretical IPv4 maximum is 65,535 bytes; practical max on Ethernet is 1500.
- Path MTU is determined dynamically via PMTUD using the DF flag and ICMP Fragmentation Needed messages.
- Use jumbo frames (9000 bytes) on dedicated internal networks for throughput gains.
- Always account for tunnel overhead (IPsec, GRE, VxLAN) which reduces effective payload size.
