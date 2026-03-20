# How to Convert IPv4 Addresses Between Binary and Decimal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Subnetting, Binary Math, IP Addressing

Description: Converting IPv4 addresses between dotted-decimal and binary notation is a fundamental subnetting skill that reveals how subnet masks, network addresses, and host ranges are calculated.

## The Dotted-Decimal and Binary Formats

An IPv4 address is 32 bits divided into four 8-bit octets. Each octet is displayed as a decimal number (0–255) separated by dots, but the underlying representation is binary.

Example: `192.168.10.5`

| Octet | Decimal | Binary |
|-------|---------|--------|
| 1 | 192 | 11000000 |
| 2 | 168 | 10101000 |
| 3 | 10  | 00001010 |
| 4 | 5   | 00000101 |

Full binary: `11000000.10101000.00001010.00000101`

## Binary to Decimal Conversion

Each bit position has a positional value (128, 64, 32, 16, 8, 4, 2, 1):

```
Bit positions: 128  64  32  16   8   4   2   1
Binary:          1   1   0   0   0   0   0   0
                128+ 64+  0+  0+  0+  0+  0+  0 = 192
```

## Python: Converting Addresses

```python
import socket
import struct

def ipv4_to_binary(ip: str) -> str:
    """Convert dotted-decimal IPv4 to binary string (no dots)."""
    return ''.join(f'{int(octet):08b}' for octet in ip.split('.'))

def binary_to_ipv4(binary: str) -> str:
    """Convert 32-bit binary string to dotted-decimal."""
    # Split into 4 octets of 8 bits each
    octets = [str(int(binary[i:i+8], 2)) for i in range(0, 32, 8)]
    return '.'.join(octets)

def ipv4_to_int(ip: str) -> int:
    """Convert dotted-decimal to a 32-bit integer."""
    return struct.unpack("!I", socket.inet_aton(ip))[0]

def int_to_ipv4(n: int) -> str:
    """Convert a 32-bit integer to dotted-decimal."""
    return socket.inet_ntoa(struct.pack("!I", n))

# Examples
ip = "192.168.10.5"
binary = ipv4_to_binary(ip)
print(f"{ip} -> {binary}")
# 192.168.10.5 -> 11000000101010000000101000000101

print(f"{binary} -> {binary_to_ipv4(binary)}")
# 11000000101010000000101000000101 -> 192.168.10.5

n = ipv4_to_int(ip)
print(f"{ip} -> {n} -> {int_to_ipv4(n)}")
# 192.168.10.5 -> 3232238085 -> 192.168.10.5
```

## Octet-by-Octet Quick Reference

| Decimal | Binary   |
|---------|----------|
| 0       | 00000000 |
| 128     | 10000000 |
| 192     | 11000000 |
| 224     | 11100000 |
| 240     | 11110000 |
| 248     | 11111000 |
| 252     | 11111100 |
| 254     | 11111110 |
| 255     | 11111111 |

These values appear frequently in subnet masks.

## Mental Conversion Trick

Break the octet into powers of 2 from left to right:
- Is the value ≥ 128? Write 1, subtract 128; else write 0.
- Is the remainder ≥ 64? Write 1, subtract 64; else write 0.
- Continue for 32, 16, 8, 4, 2, 1.

For decimal 172: 172≥128 → 1 (rem 44); 44<64 → 0; 44≥32 → 1 (rem 12); 12<16 → 0; 12≥8 → 1 (rem 4); 4≥4 → 1 (rem 0); 0<2 → 0; 0<1 → 0 = **10101100** = 172 ✓

## Key Takeaways

- Each IPv4 octet is 8 bits; convert each independently between decimal and binary.
- Binary representation is required to understand subnet masks and bitwise AND operations.
- Python's `socket.inet_aton()` and `struct.pack()` handle conversions efficiently.
- Memorize common values (128, 192, 224, 240, 252, 255) to speed up subnetting work.
