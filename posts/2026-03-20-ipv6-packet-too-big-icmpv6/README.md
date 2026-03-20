# How to Understand ICMPv6 Packet Too Big Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ICMPv6, Packet Too Big, Path MTU Discovery, RFC 4443

Description: Understand the structure and purpose of ICMPv6 Packet Too Big messages, how routers generate them, and how sources use them to implement Path MTU Discovery.

## Introduction

ICMPv6 Packet Too Big (Type 2) is the mechanism that replaces IPv4 router fragmentation. When an IPv6 router receives a packet that is too large to forward on the next link, it cannot fragment the packet - instead, it drops the packet and sends a Packet Too Big message back to the source. The source uses this information to reduce its packet size and retransmit.

## Packet Too Big Message Format

```text
ICMPv6 Packet Too Big Message (RFC 4443):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type = 2  |     Code = 0  |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             MTU                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    As much of invoking packet as possible without the ICMPv6  |
+   packet exceeding the minimum IPv6 MTU (1280 bytes)          |
```

```text
Field definitions:
  Type:  2 (Packet Too Big)
  Code:  0 (always zero for this message type)
  MTU:   The maximum MTU that the reporting router's next link can handle
         This is the value the source should use for its PMTU cache
  Body:  As much of the original (too-large) packet as possible,
         up to a total ICMPv6 message size of 1280 bytes
         (allows the source to identify which packet caused the error)
```

## How Routers Generate Packet Too Big

```text
Router PTB generation process:

1. Router receives IPv6 packet from ingress interface
2. Looks up destination in routing table
3. Selects egress interface with MTU < packet size
4. Action:
   a. Drop the original packet (cannot forward)
   b. Construct ICMPv6 Packet Too Big:
      - Src: Router's address on the ingress interface
      - Dst: Original packet's source address
      - MTU: Egress interface MTU
      - Body: Copy first (1280 - 48) = 1232 bytes of original packet
        (1280 total - 40 IPv6 - 8 ICMPv6 header)
5. Send PTB back to original source
```

## Parsing a Packet Too Big Message

```python
import struct

def parse_packet_too_big(icmpv6_data: bytes) -> dict:
    """
    Parse an ICMPv6 Packet Too Big message.
    icmpv6_data: bytes starting from the ICMPv6 header (Type byte)
    """
    if len(icmpv6_data) < 8:
        raise ValueError("ICMPv6 PTB requires at least 8 bytes")

    icmp_type, icmp_code, checksum, mtu = struct.unpack("!BBHI", icmpv6_data[:8])

    if icmp_type != 2:
        raise ValueError(f"Expected Type=2 (PTB), got Type={icmp_type}")

    # The body contains the original invoking packet
    invoking_packet = icmpv6_data[8:]

    # Parse the first 40 bytes as the IPv6 header of the original packet
    original_header = None
    if len(invoking_packet) >= 40:
        version_tc_fl = struct.unpack("!I", invoking_packet[0:4])[0]
        original_header = {
            "version": (version_tc_fl >> 28) & 0xF,
            "payload_length": struct.unpack("!H", invoking_packet[4:6])[0],
            "next_header": invoking_packet[6],
            "hop_limit": invoking_packet[7],
        }

    return {
        "type": icmp_type,
        "code": icmp_code,
        "reported_mtu": mtu,
        "effective_mtu": max(mtu, 1280),  # RFC 8200: never below 1280
        "must_use_fragment_header": mtu < 1280,
        "original_packet_preview": invoking_packet[:20].hex(),
        "original_ipv6_header": original_header,
    }
```

## The Special Case: MTU < 1280

RFC 8200 Section 5 defines special handling when the reported MTU is less than 1280:

```python
def handle_packet_too_big(reported_mtu: int, destination: str,
                           pmtu_cache: dict) -> dict:
    """
    Process a received ICMPv6 Packet Too Big message.
    Updates the PMTU cache for the destination.
    """
    if reported_mtu < 1280:
        # RFC 8200 Section 5: If notified MTU < 1280, source must still
        # send 1280-byte packets (using Fragment Header to avoid
        # further fragmentation issues on the sub-1280 link)
        effective_mtu = 1280
        requires_fragment_header = True
        note = "Sub-1280 MTU: send 1280-byte packets with Fragment Header"
    else:
        effective_mtu = reported_mtu
        requires_fragment_header = False
        note = f"Update PMTU cache to {effective_mtu}"

    pmtu_cache[destination] = {
        "mtu": effective_mtu,
        "requires_fragment_header": requires_fragment_header,
    }

    return {
        "destination": destination,
        "reported_mtu": reported_mtu,
        "effective_mtu": effective_mtu,
        "requires_fragment_header": requires_fragment_header,
        "note": note,
    }

cache = {}
print(handle_packet_too_big(1200, "2001:db8::1", cache))  # Below 1280
print(handle_packet_too_big(1400, "2001:db8::2", cache))  # Normal case
```

## Monitoring PTB Messages

```bash
# Capture all incoming PTB messages

sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2"

# Show PTB with the MTU value (bytes 44-47 of IPv6+ICMPv6)
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2" 2>&1 | \
    grep -E "packet-too-big|mtu"

# Count PTB messages received per minute
sudo tcpdump -i eth0 -q "icmp6 and ip6[40] == 2" 2>&1 | \
    awk '{count++; if (NR % 60 == 0) print count " PTBs in last minute"; }'
```

## Conclusion

ICMPv6 Packet Too Big is the cornerstone of IPv6 Path MTU Discovery. Routers generate it whenever a packet exceeds the next-link MTU, allowing sources to learn and cache the path MTU. The special rule for reported MTU values below 1280 ensures compatibility with any IPv6 link by directing the source to use Fragment Headers. The most critical operational requirement: Packet Too Big messages must never be blocked by firewalls, as this would break PMTUD for all traffic passing through that firewall.
