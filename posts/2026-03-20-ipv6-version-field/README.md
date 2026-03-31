# How to Understand the IPv6 Version Field

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Header, Networking, Protocol, Packet Structure

Description: Understand the 4-bit Version field in the IPv6 header, its value, why it matters, and how it is used to distinguish IPv6 from IPv4 packets at the packet level.

## Introduction

The first field in both IPv4 and IPv6 headers is the 4-bit Version field. For IPv6 packets, this field always contains the value `6` (binary: `0110`). While seemingly trivial, the Version field is the first thing any network device checks to determine how to parse the rest of the packet.

## Field Specification

```text
IPv6 Header (first 4 bits):
  Bits 0-3: Version = 0b0110 = 0x6 = decimal 6

IPv4 Header (first 4 bits):
  Bits 0-3: Version = 0b0100 = 0x4 = decimal 4

First byte of an IPv6 packet always starts with 0x6x:
  0x60 = Version 6, Traffic Class high nibble = 0
  0x61 = Version 6, Traffic Class high nibble = 1
  0x62 = Version 6, Traffic Class high nibble = 2
  ...

First byte of an IPv4 packet always starts with 0x4x:
  0x45 = Version 4, IHL = 5 (20 bytes, no options)
```

## Identifying IPv6 vs IPv4 from Raw Bytes

```python
def identify_ip_version(raw_packet: bytes) -> str:
    """
    Identify the IP version from raw packet bytes.

    Args:
        raw_packet: Raw bytes of the IP packet (not including L2 header)

    Returns:
        "IPv4", "IPv6", or "Unknown"
    """
    if len(raw_packet) < 1:
        return "Unknown"

    # The version is in the upper 4 bits of the first byte
    version = (raw_packet[0] >> 4) & 0xF

    if version == 4:
        return "IPv4"
    elif version == 6:
        return "IPv6"
    else:
        return f"Unknown (version {version})"

# Test with raw packet bytes

# IPv6 packet starting with 0x60 (version=6, TC high nibble=0)
ipv6_bytes = bytes([0x60, 0x00, 0x00, 0x00] + [0x00] * 36)
print(identify_ip_version(ipv6_bytes))   # IPv6

# IPv4 packet starting with 0x45 (version=4, IHL=5)
ipv4_bytes = bytes([0x45, 0x00, 0x00, 0x28] + [0x00] * 16)
print(identify_ip_version(ipv4_bytes))   # IPv4
```

## Version Field in Protocol Demultiplexing

At the network layer, the Version field is used by the OS kernel to decide which protocol handler to invoke:

```c
/* Conceptual kernel packet dispatch (simplified) */
void handle_ip_packet(uint8_t *packet, size_t len) {
    uint8_t version = (packet[0] >> 4) & 0xF;

    switch (version) {
        case 4:
            handle_ipv4_packet(packet, len);
            break;
        case 6:
            handle_ipv6_packet(packet, len);
            break;
        default:
            /* Unknown IP version, drop packet */
            drop_packet(packet, len);
            break;
    }
}
```

## Ethernet EtherType vs IP Version Field

Note that IPv4 and IPv6 are also distinguished at Layer 2 by the EtherType:

```text
Ethernet frame:
  EtherType 0x0800 = IPv4
  EtherType 0x86DD = IPv6

Once the Ethernet header is stripped, the IP Version field provides
a second layer of verification that the payload matches the EtherType.
```

```bash
# Capture only IPv6 frames using EtherType filter
sudo tcpdump -i eth0 "ether proto 0x86DD"

# Equivalently:
sudo tcpdump -i eth0 ip6

# Display the first byte of each packet (should always show 0x6x for IPv6)
sudo tcpdump -i eth0 -XX ip6 2>/dev/null | grep "0x0000" | head -10
```

## Can the Version Field Be Different From 6?

In an IPv6 packet: no. The version field is always 6. It is the first sanity check when parsing - if you receive a packet with EtherType 0x86DD but the Version field is not 6, the packet is malformed or corrupt.

```python
# Python: validate IPv6 packet version field
def validate_ipv6_version(raw_packet: bytes) -> bool:
    """Return True only if the version field is exactly 6."""
    if len(raw_packet) < 1:
        return False
    version = (raw_packet[0] >> 4) & 0xF
    if version != 6:
        print(f"Invalid IPv6 version: {version} (expected 6)")
        return False
    return True
```

## Historical Context: Other IP Versions

The Version field was designed to accommodate future IP protocol versions. The values 0-15 are reserved:

| Value | Protocol |
|---|---|
| 4 | IPv4 (RFC 791) |
| 6 | IPv6 (RFC 8200) |
| Others | Reserved or experimental |

IPv5 (ST-II, RFC 1819) was an experimental stream protocol that briefly used version 5. It was never widely deployed, but its allocation is why the next version after IPv4 is IPv6 (skipping 5).

## Conclusion

The IPv6 Version field is a 4-bit field always set to `6`, located in the most significant bits of the first byte of every IPv6 packet. It is the first piece of information any receiver uses to decide how to parse the packet. While trivial to understand, it is foundational - a corrupted or incorrect Version field makes the entire packet uninterpretable, and filtering on this field is the fastest way to separate IPv6 from IPv4 traffic at the packet level.
