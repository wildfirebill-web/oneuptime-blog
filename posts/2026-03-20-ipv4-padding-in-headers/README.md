# How to Understand IPv4 Padding in Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, IP Options, Packet Structure, TCP/IP

Description: IPv4 header padding uses zero bytes to align the total header length to a 32-bit boundary when IP Options do not fill a complete 32-bit word.

## Why Padding Is Needed

The IPv4 header length (IHL field) is measured in 32-bit (4-byte) words. When IP Options are present, their combined length may not be a multiple of 4 bytes. Padding bytes (value 0x00) are appended after the options to round up to the next 32-bit boundary.

## Padding Byte Value and Position

The NOP option (type 1) and the End of Options List option (type 0) both serve as padding:
- **NOP (0x01)**: Used between options to align individual options to convenient boundaries.
- **EOL (0x00)**: Marks the end of options; any remaining bytes in the options area are implicitly padding.

## Visualizing Header Layout with Options

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Option Type   |  Option Len   |       Option Data             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Opt Data (cont)           |  NOP (0x01)   |  EOL (0x00)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

In this example, the option occupies 6 bytes (2-byte header + 4-byte data), followed by NOP and EOL to pad to 8 bytes (two 32-bit words).

## Python: Adding Correct Padding

```python
import struct

def pad_ip_options(options: bytes) -> bytes:
    """
    Pad IP options to the next 32-bit boundary using EOL (0x00) bytes.
    """
    remainder = len(options) % 4
    if remainder != 0:
        padding_needed = 4 - remainder
        options += b'\x00' * padding_needed  # EOL bytes as padding
    return options

# Example: a 3-byte custom option (needs 1 byte of padding)

raw_option = bytes([7, 3, 0])  # Record Route skeleton (type=7, len=3)
padded = pad_ip_options(raw_option)
print(f"Options length: {len(raw_option)} -> padded: {len(padded)}")
# Options length: 3 -> padded: 4

# IHL = (20 + len(padded)) // 4
ihl = (20 + len(padded)) // 4
print(f"IHL field value: {ihl}")  # 6 (24-byte header)
```

## Verifying IHL and Padding with Scapy

```python
from scapy.all import IP, IPOption_NOP, raw

# Build a packet with NOP options as explicit padding
pkt = IP(dst="10.0.0.1", options=[IPOption_NOP(), IPOption_NOP(),
                                   IPOption_NOP(), IPOption_NOP()])
raw_bytes = raw(pkt)
ihl = (raw_bytes[0] & 0x0F) * 4
print(f"Header length with padding: {ihl} bytes")
```

## Impact on Performance

Routers must process every option byte, including padding. Packets with options are often punted from hardware forwarding to the software slow path, increasing latency by orders of magnitude. This is one reason IP options are avoided in modern networks.

## Key Takeaways

- IPv4 headers must be a multiple of 32 bits (4 bytes); padding fills any gap.
- NOP (0x01) and EOL (0x00) are the standard padding bytes.
- The IHL field reflects the total header size in 4-byte units, including padding.
- Padding is purely structural and carries no semantic meaning.
