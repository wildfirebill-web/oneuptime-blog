# How to Calculate TCP Throughput from Window Size and RTT

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Networking, Throughput, Performance, Bandwidth-Delay Product

Description: Calculate theoretical maximum TCP throughput using the bandwidth-delay product formula and validate against measured performance to identify TCP optimization opportunities.

## Introduction

TCP throughput can be precisely calculated using the bandwidth-delay product (BDP) formula. This calculation tells you the maximum throughput achievable given the current window size and RTT, letting you identify whether poor throughput is caused by TCP window limitations or actual network bandwidth constraints.

## The TCP Throughput Formula

```text
Maximum Throughput = Receive Window Size / RTT

Where:
- Receive Window Size = minimum of sender and receiver window (in bytes)
- RTT = round-trip time (in seconds)
```

## Step 1: Measure Current RTT

```bash
# Measure RTT with ping (in milliseconds)

ping -c 20 10.20.0.5 | tail -3
# rtt min/avg/max/mdev = 45.2/46.8/48.1/0.7 ms

RTT = 0.0468 seconds (46.8ms)
```

## Step 2: Find the Actual Window Size

```bash
# Check the receive window size being used in active connections
ss -tin state established | grep -i "rcv_space\|snd_wnd" | head -5

# Or capture during a transfer
tcpdump -i eth0 -n -v 'tcp port 8080' 2>/dev/null | \
  grep "win " | awk '{print $NF}' | sort -n | tail -5
# Highest window value seen during transfer = effective window

# For a specific connection, use ss
ss -tin "( dst 10.20.0.5 and dport = :8080 )"
# Look for: rcv_space:<value>
```

## Step 3: Calculate Theoretical Max

```python
def tcp_throughput_analysis(window_bytes, rtt_ms):
    """Calculate theoretical TCP throughput and compare to 1 Gbps."""
    rtt_sec = rtt_ms / 1000
    max_throughput_bytes = window_bytes / rtt_sec
    max_throughput_mbps = (max_throughput_bytes * 8) / 1_000_000

    print(f"Window size: {window_bytes/1024:.1f} KB ({window_bytes} bytes)")
    print(f"RTT: {rtt_ms} ms")
    print(f"Theoretical max throughput: {max_throughput_mbps:.1f} Mbps")
    print(f"Theoretical max throughput: {max_throughput_bytes/1024/1024:.1f} MB/s")

    # What window would we need for 1 Gbps?
    needed_for_1gbps = (1_000_000_000 / 8) * rtt_sec
    print(f"\nWindow needed for 1 Gbps with {rtt_ms}ms RTT: {needed_for_1gbps/1024/1024:.1f} MB")

# Example: 131KB window, 46ms RTT
tcp_throughput_analysis(131072, 46)
# Window size: 128.0 KB
# Theoretical max throughput: 22.8 Mbps
# Window needed for 1 Gbps: 5.7 MB
```

## Step 4: Measure Actual Throughput and Compare

```bash
# Measure actual throughput with iperf3
iperf3 -c 10.20.0.5 -t 10

# Compare to calculated max
# If actual ≈ calculated: window is the bottleneck (increase buffer sizes)
# If actual << calculated: congestion or packet loss is the bottleneck
```

## Step 5: Adjust Buffers to Remove Bottleneck

```bash
# If window is the bottleneck, increase socket buffers
# Needed buffer = BDP = bandwidth × RTT

# For 1 Gbps link with 50ms RTT:
# BDP = 125,000,000 bytes/sec × 0.050 sec = 6.25 MB

sysctl -w net.ipv4.tcp_rmem="4096 1048576 8388608"   # max = 8MB
sysctl -w net.ipv4.tcp_wmem="4096 1048576 8388608"
sysctl -w net.core.rmem_max=8388608
sysctl -w net.core.wmem_max=8388608

# Re-run iperf3 to verify improvement
iperf3 -c 10.20.0.5 -t 10
```

## Conclusion

The BDP formula is a powerful diagnostic tool. If your calculated maximum throughput is much lower than your network's capacity, increasing TCP buffer sizes will directly improve performance. If actual throughput is much lower than the calculated maximum, packet loss or congestion is limiting you - no amount of buffer tuning will help there. Always measure both to direct your optimization effort correctly.
