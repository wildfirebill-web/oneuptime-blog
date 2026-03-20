# How to Understand Why IPv6 Removed the Header Checksum

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Headers, Checksums, Performance, Protocol Design

Description: Understand the reasoning behind IPv6's decision to remove the header checksum present in IPv4, and how data integrity is maintained at other layers.

## Introduction

IPv4 includes a 16-bit header checksum that every router must verify and recalculate when it decrements the TTL. IPv6 removes this checksum entirely. This was a controversial but deliberate design decision that improves router performance. Understanding the rationale requires examining how data integrity is maintained without a header checksum.

## Why IPv4 Had a Header Checksum

IPv4 was designed in the 1970s when network hardware was unreliable. Bit errors in transit were common. The header checksum provided a fast way to detect if the header had been corrupted during transmission, preventing misrouted packets due to corrupted destination addresses.

## Why IPv6 Removed It

### 1. Every Router Must Recalculate It

In IPv4, the TTL field changes at every hop. Since the checksum covers the entire header (including TTL), it must be recalculated at every router:

```text
IPv4 router processing:
  1. Verify header checksum (read all 20 bytes, compute checksum)
  2. Decrement TTL
  3. Recalculate header checksum (compute checksum of modified header)
  4. Forward packet

This adds 2 checksum operations per hop, per packet.
At 10 Gbps, this is millions of operations per second.
```

### 2. Link-Layer Protection Makes It Redundant

Modern network technologies all provide their own error detection:

```text
Ethernet: 32-bit CRC (Frame Check Sequence)
  → Catches all single-bit errors and most burst errors
  → Verified by every NIC, every switch, every router

802.11 WiFi: CRC-32 per frame

Fiber optic links: 8b/10b or 64b/66b encoding with FEC
  → Forward Error Correction means fewer bit errors reach the IP layer

Result: By the time an IPv6 packet is processed,
        the link-layer CRC has already verified the bits.
```

### 3. End-to-End Checksums Are Mandatory

IPv6 made TCP and UDP checksums mandatory (UDP checksum is optional in IPv4 but required in IPv6):

```python
# IPv6 mandates checksum in all upper-layer protocols:

# TCP:    Checksum required (was already required in IPv4)
# UDP:    Checksum required (optional in IPv4 over IPv4)
# ICMPv6: Checksum required (optional in IPv4)
# SCTP:   Checksum required

# These checksums cover:
# - Upper-layer data
# - Upper-layer header
# - IPv6 pseudo-header (src, dst, length, next header)
# So any bit error in the IPv6 header addresses will be caught by
# the transport-layer checksum
```

## Performance Impact

```python
def estimate_checksum_savings(
    packet_rate_mpps: float,  # Millions of packets per second
    routers_in_path: int = 10
) -> dict:
    """
    Estimate the per-router computational savings from removing the header checksum.
    """
    # IPv4: 2 checksum operations per hop (verify + recalculate)
    # Each checksum operation: ~20 bytes = ~5 32-bit words
    # Assume each 32-bit add/XOR = 1 CPU cycle

    ops_per_packet_ipv4 = 2 * 5  # verify + recalculate, ~5 words each
    ops_per_packet_ipv6 = 0       # no header checksum

    total_packets_per_second = packet_rate_mpps * 1_000_000

    ipv4_ops = total_packets_per_second * ops_per_packet_ipv4
    ipv6_ops = total_packets_per_second * ops_per_packet_ipv6

    return {
        "packet_rate": f"{packet_rate_mpps} Mpps",
        "ipv4_checksum_ops_per_sec": f"{ipv4_ops:,.0f}",
        "ipv6_checksum_ops_per_sec": f"{ipv6_ops:,.0f}",
        "savings_percent": 100 - (ipv6_ops / ipv4_ops * 100) if ipv4_ops > 0 else 100,
    }

result = estimate_checksum_savings(100)  # 100 Mpps (a real core router rate)
print(f"IPv4 header checksum ops/sec: {result['ipv4_checksum_ops_per_sec']}")
print(f"IPv6 header checksum ops/sec: {result['ipv6_checksum_ops_per_sec']}")
print(f"Savings: {result['savings_percent']:.0f}%")
```

## The One Risk: Silent Misrouting

The legitimate concern about removing the header checksum is **silent misrouting**:

```text
Scenario without header checksum:
1. A bit error corrupts a destination address in the IPv6 header
2. The link-layer CRC does NOT catch it (bit error after CRC calculation)
3. The packet is delivered to the wrong destination
4. No error is returned to the sender

However, the end-to-end checksum will cause the misdelivered packet
to be silently discarded by the wrong recipient.
The sender's TCP retransmission will eventually recover the data.
```

The IPv6 designers accepted this tradeoff: silent packet loss is acceptable; the upper-layer protocols (TCP) handle retransmission.

## Conclusion

IPv6's removal of the header checksum was a deliberate tradeoff: accept occasional silent packet loss in exchange for dramatic router performance improvements. The rationale is sound - link-layer error detection has become reliable, upper-layer checksums provide end-to-end protection, and UDP checksums are mandatory in IPv6. The result is that routers do not need to perform per-hop checksumming, enabling significantly higher forwarding rates in hardware.
