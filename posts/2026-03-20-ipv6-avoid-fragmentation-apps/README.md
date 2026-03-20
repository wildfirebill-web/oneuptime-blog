# How to Avoid IPv6 Fragmentation in Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fragmentation, Application Design, PMTUD, Socket Options

Description: Design IPv6 applications to avoid fragmentation by using PMTU Discovery, setting appropriate socket options, and keeping packets within the path MTU.

## Introduction

IPv6 fragmentation should be a last resort. Fragmented packets are more likely to be dropped by middleboxes, add reassembly overhead at the destination, and can fail silently if reassembly buffers are exhausted. Well-designed IPv6 applications use Path MTU Discovery or conservative packet sizing to avoid fragmentation entirely.

## Strategies to Avoid Fragmentation

```
Anti-fragmentation strategies ranked by preference:

1. Use TCP (handles PMTUD and MSS automatically)
   → TCP negotiates MSS, adjusts to path MTU dynamically
   → No application-level effort needed

2. Keep packets below minimum IPv6 MTU (1280 bytes)
   → Guaranteed to work on all IPv6 paths
   → Limits throughput on high-MTU paths
   → Appropriate for DNS queries, NDP, ICMPv6

3. Use PMTU Discovery with UDP (RFC 8899 for QUIC/DTLS)
   → Application handles Packet Too Big and reduces packet size
   → More complex but allows full path MTU utilization

4. Set IP_DONTFRAGMENT socket option
   → Get explicit errors instead of silent fragmentation
   → Allows application to handle MTU feedback

5. Fragment at application layer (not IPv6 level)
   → Application splits messages into MTU-sized chunks
   → No IPv6 Fragment Headers needed
```

## Socket Options for Controlling Fragmentation

```python
import socket
import struct

def create_udp6_no_fragment_socket() -> socket.socket:
    """
    Create an IPv6 UDP socket that disables fragmentation
    and returns errors when packets exceed the path MTU.
    """
    s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)

    # IPV6_DONTFRAG: If set, send returns EMSGSIZE for over-MTU packets
    # instead of silently fragmenting
    IPV6_DONTFRAG = 62  # Linux constant
    s.setsockopt(socket.IPPROTO_IPV6, IPV6_DONTFRAG, 1)

    # IPV6_RECVPATHMTU: Request PMTU change notifications
    IPV6_RECVPATHMTU = 60  # Linux constant
    s.setsockopt(socket.IPPROTO_IPV6, IPV6_RECVPATHMTU, 1)

    # IPV6_USE_MIN_MTU: Set to 1 to use 1280-byte minimum for all sends
    # Useful for applications that need guaranteed delivery without PMTUD
    # IPV6_USE_MIN_MTU = 63
    # s.setsockopt(socket.IPPROTO_IPV6, IPV6_USE_MIN_MTU, 1)

    return s

def send_with_mtu_awareness(s: socket.socket, data: bytes,
                             dest: tuple, current_pmtu: int = 1500) -> int:
    """
    Send data, handling EMSGSIZE errors by reducing packet size.
    dest: (address, port)
    Returns actual bytes sent.
    """
    max_payload = current_pmtu - 40 - 8  # IPv6 header + UDP header

    if len(data) <= max_payload:
        try:
            return s.sendto(data, dest)
        except OSError as e:
            if e.errno == 90:  # EMSGSIZE: packet too large
                print(f"Packet too large for current PMTU={current_pmtu}")
                return -1
            raise
    else:
        # Application-layer chunking (preferred over IPv6 fragmentation)
        sent = 0
        for i in range(0, len(data), max_payload):
            chunk = data[i:i + max_payload]
            s.sendto(chunk, dest)
            sent += len(chunk)
        return sent
```

## DNS as a Case Study

DNS is a real-world example of fragmentation avoidance strategy:

```bash
# DNS uses UDP with max 512 bytes traditionally (fits in any MTU)
# EDNS0 extends to 4096 bytes with fragmentation risk

# Check if DNS responses require fragmentation
# Large DNSSEC responses can exceed 1280 bytes
dig @8.8.8.8 cloudflare.com DNSKEY +dnssec +bufsize=1232

# 1232 = 1280 (min IPv6 MTU) - 40 (IPv6) - 8 (UDP)
# Setting bufsize=1232 avoids fragmentation on all IPv6 paths

# Modern recommendation (RFC 8900): keep DNS UDP responses ≤ 1232 bytes
# If response is larger, truncate and let client retry over TCP

# Configure BIND to limit UDP response size
# In named.conf options:
# edns-udp-size 1232;
# max-udp-size 1232;
```

## QUIC and Modern Protocols

```
Modern protocol approaches to avoid fragmentation:

QUIC (RFC 9000):
  - Runs over UDP
  - Built-in PMTU Discovery (RFC 9000 Section 14)
  - Uses PADDING frames to probe MTU
  - Falls back to 1280-byte minimum if PMTUD fails
  - No application-level fragmentation needed

DTLS (Datagram TLS):
  - Applications split records to fit PMTU
  - PMTU Discovery handled per RFC 6347

WireGuard:
  - Sets tunnel MTU = underlying MTU - 60 bytes (overhead)
  - All packets fit within outer link MTU automatically
  - No fragmentation at IPsec/WireGuard layer
```

## Conclusion

The best way to avoid IPv6 fragmentation is to use TCP for connections that need it (TCP handles PMTUD automatically through MSS negotiation) and to keep UDP payloads conservative (within 1280 bytes for guaranteed delivery, or implementing application-level PMTUD for higher efficiency). The `IPV6_DONTFRAG` socket option is invaluable during development — it surfaces MTU problems as explicit errors rather than silent fragmentation failures. For DNS specifically, RFC 8900 recommends limiting UDP response sizes to 1232 bytes to avoid fragmentation on all IPv6 paths.
