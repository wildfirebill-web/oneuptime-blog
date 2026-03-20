# How to Understand the Source/Target Link-Layer Address Option in NDP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Link-Layer Address, Source Link-Layer, Target Link-Layer, MAC Address

Description: Understand the Source Link-Layer Address (SLLA) and Target Link-Layer Address (TLLA) NDP options, when they are included, and how they enable efficient neighbor discovery.

## Introduction

The Source Link-Layer Address (SLLA, Type 1) and Target Link-Layer Address (TLLA, Type 2) NDP options carry the MAC address of the sending node. These options allow the receiver to update its neighbor cache directly from the NDP message, avoiding the need for a separate NS/NA exchange. SLLA appears in Router Solicitations, Router Advertisements, and Neighbor Solicitations. TLLA appears in Neighbor Advertisements and Redirect messages.

## Option Format

```text
Source Link-Layer Address (Type 1) and Target Link-Layer Address (Type 2):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=1/2  |    Length=1   |  Link-Layer Address (6 bytes)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Link-Layer Address (continued)                      |
+-+-+-+-+-+-+-+-+-+-+-+-

Total: 8 bytes (Length=1 unit of 8 bytes)
MAC address: bytes 2-7 of the option (6 bytes for Ethernet)

For non-Ethernet links (e.g., FDDI, ATM): MAC length differs
The Length field adjusts accordingly for non-6-byte addresses
```

## When Each Option Is Included

```text
Source Link-Layer Address (Type 1):
  In NS: MUST be included (unless NS source is :: for DAD)
         Reason: Allows target to update neighbor cache without separate NA
  In RS: SHOULD be included (allows router to respond unicast)
  In RA: SHOULD be included (hosts can cache router's MAC)

  NOT in DAD NS: Source address is :: (no link-layer address known yet
                 for the tentative address being checked)

Target Link-Layer Address (Type 2):
  In NA: SHOULD be included (if sent in response to NS with SLLA option)
         Reason: Primary purpose of NA is to provide the MAC
  In Redirect: SHOULD be included (router telling host the next-hop MAC)

Why "SHOULD" not "MUST":
  If the link-layer address has already been sent recently and
  is cached, the option can be omitted to save space
```

## Parsing SLLA/TLLA Options

```python
import struct

def parse_slla_tlla(option_data: bytes) -> dict:
    """
    Parse Source or Target Link-Layer Address NDP option.

    Args:
        option_data: Complete option bytes starting from Type byte

    Returns:
        Parsed option fields
    """
    if len(option_data) < 8:
        raise ValueError("Link-Layer Address option requires 8 bytes minimum")

    opt_type = option_data[0]
    opt_len_units = option_data[1]
    opt_len_bytes = opt_len_units * 8

    # MAC address is in bytes 2-7 (6 bytes for Ethernet)
    mac_bytes = option_data[2:8]
    mac_address = ':'.join(f'{b:02x}' for b in mac_bytes)

    return {
        "type": opt_type,
        "type_name": "Source Link-Layer Address" if opt_type == 1 else "Target Link-Layer Address",
        "length_units": opt_len_units,
        "length_bytes": opt_len_bytes,
        "mac_address": mac_address,
        "raw_mac": mac_bytes,
    }

def build_slla_option(mac_address: str) -> bytes:
    """Build a Source Link-Layer Address option."""
    mac_bytes = bytes(int(x, 16) for x in mac_address.split(':'))
    if len(mac_bytes) != 6:
        raise ValueError("Ethernet MAC must be 6 bytes")
    return struct.pack("!BB", 1, 1) + mac_bytes

def build_tlla_option(mac_address: str) -> bytes:
    """Build a Target Link-Layer Address option."""
    mac_bytes = bytes(int(x, 16) for x in mac_address.split(':'))
    return struct.pack("!BB", 2, 1) + mac_bytes

# Example

slla = build_slla_option("00:11:22:33:44:55")
parsed = parse_slla_tlla(slla)
print(f"SLLA: type={parsed['type_name']}, MAC={parsed['mac_address']}")

tlla = build_tlla_option("aa:bb:cc:dd:ee:ff")
parsed = parse_slla_tlla(tlla)
print(f"TLLA: type={parsed['type_name']}, MAC={parsed['mac_address']}")
```

## Neighbor Cache Update from SLLA/TLLA

```text
When a node receives NDP with SLLA/TLLA:

Receiving NS with SLLA:
  → Create or update neighbor cache for NS sender
  → NS sender's IPv6 source address → SLLA MAC
  → This avoids the need for the target to send a separate NS
    to learn the requester's MAC

Receiving NA with TLLA:
  → Update neighbor cache for NA's target address
  → Target address → TLLA MAC
  → Move state to REACHABLE (if S flag set in NA)

Receiving RA with SLLA:
  → Update neighbor cache for router (fe80::...)
  → Router's link-local address → SLLA MAC
  → Can immediately forward packets to router without NS/NA
```

## Conclusion

Source and Target Link-Layer Address options are the mechanism by which NDP carries MAC address information alongside the NS/NA exchange. Including SLLA in an NS allows the target to update its neighbor cache for the requester without needing to send its own NS. Including TLLA in an NA provides the MAC address in the same packet that confirms reachability. These options are what make NDP an integrated mechanism rather than a two-phase lookup protocol.
