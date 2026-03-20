# How TCP Handles IPv6 Jumbograms with MSS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TCP, Jumbograms, MSS, Maximum Segment Size

Description: Understand how TCP Maximum Segment Size interacts with IPv6 jumbograms, how TCP can send segments larger than 65495 bytes, and the practical limits of TCP over jumbogram paths.

## Introduction

TCP's Maximum Segment Size (MSS) is negotiated during the SYN handshake and defines the maximum amount of data per TCP segment. On standard paths, MSS is capped at 65495 bytes by the 16-bit IPv6 Payload Length field (65535 - 40 IPv6 header). On jumbogram-capable paths where the link MTU exceeds 65535 bytes, TCP can theoretically use larger MSS values, allowing a single TCP segment to carry more than 65535 bytes of application data.

## TCP MSS Standard Limits

```
Standard TCP MSS calculation:

IPv6 payload length limit: 65535 bytes (16-bit field)
  TCP header: minimum 20 bytes
  TCP MSS max: 65535 - 20 = 65515 bytes per segment

With TCP options (12 bytes of options):
  TCP MSS max: 65535 - 32 = 65503 bytes

IPv6 header is NOT counted in IPv6 Payload Length:
  So TCP MSS = 65535 - TCP_header_size (not - 40)

In practice, standard Ethernet limits MSS to:
  1500 - 40 (IPv6) - 20 (TCP) = 1440 bytes
```

## TCP MSS with Jumbograms

```
Jumbogram TCP MSS:

On a jumbogram path (link MTU > 65535):
  IPv6 Payload Length = 0 (jumbogram indicator)
  Actual length in Jumbo Payload Hop-by-Hop option (32 bits)
  Maximum TCP segment: up to ~4 GB theoretically

TCP sequence number space:
  TCP uses 32-bit sequence numbers
  Max unacknowledged data: ~4 GB (before wrap-around)
  SACK allows efficient handling of out-of-order segments
  No change to TCP protocol needed for jumbogram support

MSS negotiation for jumbograms:
  MSS is a 16-bit field in TCP options
  Maximum advertised MSS: 65535 bytes
  For larger segments on jumbogram paths: MSS option value wraps or
  implementations use other mechanisms (implementation-specific)
```

## TCP Performance on Jumbogram Paths

```python
def estimate_tcp_performance(mtu: int, rtt_ms: float,
                              bandwidth_gbps: float) -> dict:
    """
    Estimate TCP throughput for a given MTU, RTT, and link bandwidth.
    Demonstrates the efficiency improvement of jumbograms.
    """
    tcp_header = 20
    ipv6_header = 40

    # MSS: maximum TCP payload per segment
    mss = mtu - ipv6_header - tcp_header

    # Bandwidth-delay product (BDP) in bytes
    bdp_bytes = (bandwidth_gbps * 1e9 / 8) * (rtt_ms / 1000)

    # Number of segments needed to fill the BDP
    segments_to_fill_bdp = bdp_bytes / mss

    # Header overhead ratio
    header_bytes_per_segment = ipv6_header + tcp_header
    payload_bytes_per_segment = mss
    overhead_percent = (header_bytes_per_segment / mtu) * 100

    return {
        "mtu": mtu,
        "mss": mss,
        "rtt_ms": rtt_ms,
        "bandwidth_gbps": bandwidth_gbps,
        "bdp_bytes": int(bdp_bytes),
        "segments_to_fill_bdp": int(segments_to_fill_bdp),
        "header_overhead_percent": round(overhead_percent, 2),
    }

# Compare standard MTU vs jumbo frames vs theoretical jumbogram
scenarios = [
    (1500,  0.1, 10),   # Standard Ethernet, 0.1ms RTT, 10 Gbps
    (9000,  0.1, 10),   # Jumbo frames, 0.1ms RTT, 10 Gbps
    (65535, 0.1, 100),  # Maximum standard, 0.1ms RTT, 100 Gbps (HPC)
]

print(f"{'MTU':<8} {'MSS':<8} {'Overhead%':<12} {'Segments for BDP'}")
print("-" * 50)
for mtu, rtt, bw in scenarios:
    r = estimate_tcp_performance(mtu, rtt, bw)
    print(f"{r['mtu']:<8} {r['mss']:<8} {r['header_overhead_percent']:<12} {r['segments_to_fill_bdp']}")
```

## Configuring TCP for Large Segment Offload

```bash
# Large Segment Offload (LSO/TSO) allows the NIC to segment large TCP buffers
# This is related to but distinct from jumbograms

# Check TSO (TCP Segmentation Offload) status
ethtool -k eth0 | grep -i "tcp-segmentation-offload\|tso"

# Enable TSO (hardware splits large TCP segments into MTU-sized frames)
sudo ethtool -K eth0 tso on

# Check GSO (Generic Segmentation Offload)
ethtool -k eth0 | grep gso

# Enable/disable for testing
sudo ethtool -K eth0 gso off  # Test without GSO
sudo ethtool -K eth0 gso on   # Re-enable

# Check TCP congestion control (important for large BDP paths)
cat /proc/sys/net/ipv4/tcp_congestion_control
# BBR or CUBIC are recommended for high-BDP paths
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## TCP Tuning for HPC Jumbogram Paths

```bash
# Increase TCP buffers for high-BDP paths with large MTU
# BDP = 100 Gbps × 1ms = 12.5 MB
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"

# Enable TCP window scaling (required for BDP > 65535 bytes)
sudo sysctl -w net.ipv4.tcp_window_scaling=1

# Set congestion control to BBR for HPC
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# Persist changes
sudo sysctl -p
```

## Conclusion

TCP handles jumbogram paths naturally because its sequence number space (32-bit) is already large enough for any payload size. The 16-bit MSS field in TCP options limits the negotiated per-segment size to 65535 bytes, but on jumbogram paths, TCP implementations can send larger segments when the path allows it. The primary benefit in practice comes from jumbo frames (9000-byte MTU) rather than true jumbograms, as 9000-byte frames are readily available on modern data center hardware and reduce TCP/IP processing overhead significantly for bulk transfers.
