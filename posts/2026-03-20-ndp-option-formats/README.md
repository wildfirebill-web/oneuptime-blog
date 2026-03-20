# How to Understand NDP Option Formats

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Options, TLV, IPv6, RFC 4861

Description: Understand the TLV (Type-Length-Value) format of NDP options, the padding rules, and how to parse and build NDP options in code.

## Introduction

NDP options use a Type-Length-Value (TLV) format where Length is in units of 8 bytes. This constraint means all options must be padded to multiples of 8 bytes. Options appear after the fixed-size NDP message body and can vary by message type. Understanding the option format is necessary for implementing NDP or parsing NDP messages in packet analysis tools.

## NDP Option General Format

```
NDP Option format:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |    Length     |   Option-specific data ...    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Type:   1 byte, identifies the option type
Length: 1 byte, LENGTH IN UNITS OF 8 BYTES (not bytes!)
        Length = 0 is invalid
        Length = 1 means 8 bytes total (including Type and Length bytes)
        Length = 2 means 16 bytes total
Data:   (Length × 8) - 2 bytes of option-specific data

Known option types:
  1: Source Link-Layer Address
  2: Target Link-Layer Address
  3: Prefix Information
  4: Redirected Header
  5: MTU
  25: RDNSS (Recursive DNS Server, RFC 8106)
  31: DNS Search List (DNSSL, RFC 8106)
```

## Option Padding Rules

```python
import struct
import math

def option_length_units(data_bytes: int) -> int:
    """
    Calculate the Length field value (in 8-byte units) for an NDP option.
    Includes 2 bytes for Type and Length fields.
    """
    total_bytes = 2 + data_bytes  # Type (1) + Length (1) + data
    # Round up to nearest multiple of 8
    padded_bytes = math.ceil(total_bytes / 8) * 8
    return padded_bytes // 8

def build_ndp_option(option_type: int, data: bytes) -> bytes:
    """
    Build an NDP option with proper padding.
    """
    length_units = option_length_units(len(data))
    total_bytes = length_units * 8
    padding_needed = total_bytes - 2 - len(data)

    return (struct.pack("!BB", option_type, length_units) +
            data +
            b'\x00' * padding_needed)

def parse_ndp_options(options_data: bytes) -> list:
    """
    Parse a sequence of NDP options.
    Returns list of (type, length_units, data) tuples.
    """
    options = []
    offset = 0

    while offset + 2 <= len(options_data):
        opt_type = options_data[offset]
        opt_len_units = options_data[offset + 1]

        if opt_len_units == 0:
            break  # Invalid: length must be > 0

        opt_len_bytes = opt_len_units * 8
        opt_data = options_data[offset + 2:offset + opt_len_bytes]

        options.append({
            "type": opt_type,
            "length_units": opt_len_units,
            "length_bytes": opt_len_bytes,
            "data": opt_data,
        })
        offset += opt_len_bytes

    return options

# Example: build Source Link-Layer Address option
mac_bytes = bytes([0x00, 0x11, 0x22, 0x33, 0x44, 0x55])
slla_option = build_ndp_option(1, mac_bytes)
print(f"SLLA option: {slla_option.hex()} ({len(slla_option)} bytes)")
# Expected: 01 01 00:11:22:33:44:55 = 8 bytes (length=1 unit)

# Parse it back
parsed = parse_ndp_options(slla_option)
print(f"Parsed: type={parsed[0]['type']}, length={parsed[0]['length_units']} units")
```

## Options by Message Type

```
Options valid in each NDP message:

Router Solicitation (Type 133):
  Type 1: Source Link-Layer Address (SHOULD be included if known)

Router Advertisement (Type 134):
  Type 1: Source Link-Layer Address (SHOULD be included)
  Type 3: Prefix Information (one per prefix; MUST be included for SLAAC)
  Type 5: MTU (SHOULD be included on links with variable MTU)
  Type 25: RDNSS (RFC 8106)
  Type 31: DNSSL (RFC 8106)

Neighbor Solicitation (Type 135):
  Type 1: Source Link-Layer Address (MUST include unless source is ::)

Neighbor Advertisement (Type 136):
  Type 2: Target Link-Layer Address (SHOULD include unless already known)

Redirect (Type 137):
  Type 2: Target Link-Layer Address
  Type 4: Redirected Header
```

## Conclusion

NDP option format is straightforward: Type (1 byte), Length in 8-byte units (1 byte), data (padded to complete the 8-byte unit). The key rule is that Length=1 means 8 bytes total, not 1 byte. Options must be padded to multiples of 8 bytes. Unknown NDP option types are silently ignored by implementations following RFC 4861, unlike extension header option types which have configurable error behaviors. The most commonly implemented options are the Source/Target Link-Layer Address, Prefix Information, and MTU options.
