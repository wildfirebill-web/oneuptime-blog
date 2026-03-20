# How to Build a Raw Socket Application for IPv4 in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Raw Socket, ICMP, Networking, Linux

Description: Learn how to work with raw IPv4 sockets in Python to craft and send custom packets, implement ICMP ping, and receive raw IP traffic for network analysis purposes.

## What Are Raw Sockets?

Raw sockets (`SOCK_RAW`) bypass the transport layer and give direct access to IP packets. They require root/administrator privileges on Linux/macOS.

```
Normal Socket:    App ↔ TCP/UDP ↔ IP ↔ Ethernet
Raw Socket:       App           ↔ IP ↔ Ethernet
```

## ICMP Ping with Raw Sockets

```python
import socket
import struct
import time
import os

def checksum(data: bytes) -> int:
    """Internet checksum (RFC 1071)."""
    if len(data) % 2:
        data += b"\x00"
    total = 0
    for i in range(0, len(data), 2):
        total += (data[i] << 8) + data[i+1]
    total  = (total >> 16) + (total & 0xFFFF)
    total += (total >> 16)
    return ~total & 0xFFFF

def build_icmp_echo(seq: int = 1) -> bytes:
    """Build an ICMP Echo Request packet."""
    pid     = os.getpid() & 0xFFFF
    header  = struct.pack("!BBHHH", 8, 0, 0, pid, seq)  # type, code, cksum, id, seq
    payload = struct.pack("!d", time.monotonic())         # 8-byte timestamp
    cksum   = checksum(header + payload)
    header  = struct.pack("!BBHHH", 8, 0, cksum, pid, seq)
    return header + payload

def ping(dest_ip: str, count: int = 4, timeout: float = 2.0) -> None:
    # IPPROTO_ICMP requires root privileges
    with socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP) as s:
        s.settimeout(timeout)
        pid = os.getpid() & 0xFFFF

        for seq in range(1, count + 1):
            packet = build_icmp_echo(seq)
            send_time = time.monotonic()
            s.sendto(packet, (dest_ip, 0))

            try:
                while True:
                    data, addr = s.recvfrom(1024)
                    recv_time = time.monotonic()
                    # Skip IP header (first 20 bytes)
                    icmp_header = data[20:28]
                    icmp_type, _, _, recv_pid, recv_seq = struct.unpack("!BBHHH", icmp_header)
                    if icmp_type == 0 and recv_pid == pid and recv_seq == seq:
                        rtt = (recv_time - send_time) * 1000
                        print(f"Reply from {addr[0]}: seq={seq} time={rtt:.2f} ms")
                        break
            except socket.timeout:
                print(f"Request timeout for seq={seq}")

# Must run as root / Administrator
# ping("8.8.8.8")
```

## Receive Raw IP Packets (Sniffer)

```python
import socket
import struct
import ipaddress

def parse_ip_header(data: bytes) -> dict:
    """Parse a 20-byte IPv4 header."""
    fields = struct.unpack("!BBHHHBBH4s4s", data[:20])
    return {
        "version":  (fields[0] >> 4),
        "ihl":      (fields[0] & 0x0F) * 4,
        "tos":      fields[1],
        "length":   fields[2],
        "id":       fields[3],
        "flags":    (fields[4] >> 13),
        "fragment": (fields[4] & 0x1FFF),
        "ttl":      fields[5],
        "protocol": fields[6],
        "checksum": fields[7],
        "src":      socket.inet_ntoa(fields[8]),
        "dst":      socket.inet_ntoa(fields[9]),
    }

PROTO_NAMES = {1: "ICMP", 6: "TCP", 17: "UDP"}

# Root required; on Linux only
with socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_IP) as s:
    # Bind to interface
    s.bind(("0.0.0.0", 0))
    # Include IP header in received data
    s.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)

    print("Sniffing IP packets (Ctrl+C to stop)")
    for _ in range(20):
        data, addr = s.recvfrom(65535)
        hdr = parse_ip_header(data)
        proto = PROTO_NAMES.get(hdr["protocol"], str(hdr["protocol"]))
        print(f"{hdr['src']:>16} → {hdr['dst']:>16}  {proto}  TTL={hdr['ttl']}")
```

## Conclusion

Raw sockets give direct access to IP-layer packets but require elevated privileges. Use them for diagnostic tools (ping, traceroute), network monitors, and custom protocol implementations. For IPv4 sniffing, `socket.IPPROTO_IP` with `IP_HDRINCL` includes the IP header in received data. For production network analysis, prefer libraries like Scapy or PyShark which handle platform differences and provide higher-level packet parsing. Never use raw sockets to forge source IPs for malicious purposes — most ISPs implement ingress filtering (BCP38) that drops spoofed packets.
