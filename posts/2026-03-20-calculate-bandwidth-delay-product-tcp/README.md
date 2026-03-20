# How to Calculate the Bandwidth Delay Product (BDP) for TCP Tuning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, BDP, Performance, Network Tuning, Sysctl, Bandwidth

Description: Learn how to calculate the Bandwidth Delay Product (BDP) for any network path and use it to correctly size TCP socket buffers for maximum throughput.

## What Is the Bandwidth Delay Product?

The Bandwidth Delay Product (BDP) is the amount of data that can be "in flight" on a network path at any given time - the product of bandwidth and round-trip time:

```text
BDP = Bandwidth (bits/sec) × Round-Trip Time (seconds)
```

TCP must buffer this much data to keep the pipe full. If the TCP buffer is smaller than the BDP, the sender will run out of buffer space before the ACK returns, causing TCP to stall and throttle throughput.

## Step 1: Measure Round-Trip Time

```bash
# Measure RTT to a target host

ping -c 20 192.168.1.100
# Output:
# rtt min/avg/max/mdev = 0.312/0.441/0.612/0.087 ms

# Use the average RTT for BDP calculation
# Here: 0.441 ms = 0.000441 seconds

# For WAN paths, use a longer ping to get stable average
ping -c 100 8.8.8.8
# rtt min/avg/max/mdev = 12.4/14.2/18.9/1.2 ms
```

## Step 2: Calculate BDP

```text
BDP = Bandwidth × RTT

Examples:

1 Gbps LAN, 0.5ms RTT:
BDP = 1,000,000,000 bits/sec × 0.0005 sec = 500,000 bits = 62,500 bytes ≈ 61 KB

10 Gbps LAN, 0.1ms RTT:
BDP = 10,000,000,000 × 0.0001 = 1,000,000 bits = 125,000 bytes ≈ 122 KB

1 Gbps WAN, 50ms RTT:
BDP = 1,000,000,000 × 0.050 = 50,000,000 bits = 6,250,000 bytes ≈ 6 MB

10 Gbps WAN, 100ms RTT:
BDP = 10,000,000,000 × 0.100 = 1,000,000,000 bits = 125,000,000 bytes ≈ 119 MB
```

A quick bash calculator:

```bash
# BDP calculator (in bytes)
bandwidth_gbps=10
rtt_ms=50
bdp_bytes=$(echo "scale=0; $bandwidth_gbps * 1000000000 * $rtt_ms / 1000 / 8" | bc)
echo "BDP = $bdp_bytes bytes = $(echo "$bdp_bytes / 1048576" | bc) MB"
# BDP = 62500000 bytes = 59 MB
```

## Step 3: Determine Required Buffer Size

Set TCP buffers to at least **2× BDP** to provide headroom:

```text
Required buffer = 2 × BDP

For 10 Gbps WAN, 100ms RTT:
BDP = 125 MB
Required buffer = 250 MB
```

The factor of 2 accounts for:
- One BDP of data in flight from sender to receiver
- One BDP of ACKs traveling in the reverse direction
- Jitter and burst headroom

## Step 4: Set TCP Buffers to Match BDP

```bash
# Example: 10 Gbps link with 50ms RTT
# BDP = 59 MB, required buffer = 128 MB (round up to power of 2)

sudo sysctl -w net.core.rmem_max=268435456      # 256 MB
sudo sysctl -w net.core.wmem_max=268435456
sudo sysctl -w net.ipv4.tcp_rmem="4096 1048576 268435456"
sudo sysctl -w net.ipv4.tcp_wmem="4096 1048576 268435456"
```

## Step 5: Verify TCP Is Using the Full Window

After tuning, verify that TCP is actually using large windows:

```bash
# Check active connection window sizes
ss -tin | grep rcv_space

# Look for "rcv_space:XXXXX" - should be close to your max buffer
# Also check:
ss -tin | grep -E "wscale|snd_wnd|rcv_wnd"
```

Run an iperf3 test and compare before/after:

```bash
# Before tuning (default buffers)
iperf3 -c server-ip -t 30

# After setting BDP-sized buffers
iperf3 -c server-ip -t 30
# Throughput should increase significantly on high-BDP paths
```

## Step 6: BDP for Common Network Scenarios

| Link | RTT | BDP | Required Buffer |
|---|---|---|---|
| 1G LAN | 0.5 ms | 62 KB | 1 MB |
| 10G LAN | 0.1 ms | 12.5 KB | 1 MB |
| 1G WAN (regional) | 20 ms | 2.5 MB | 8 MB |
| 1G WAN (cross-country) | 80 ms | 10 MB | 32 MB |
| 10G WAN (cross-country) | 80 ms | 100 MB | 256 MB |
| Satellite | 600 ms | 75 MB | 256 MB |

## Conclusion

The BDP calculation - bandwidth multiplied by RTT - tells you exactly how large TCP buffers must be to fill a network pipe. Use `ping` to measure RTT, multiply by your link bandwidth, double it for safety margin, and set `net.ipv4.tcp_rmem` and `tcp_wmem` max values accordingly. For LAN connections the default buffers are usually adequate; for high-latency WAN links, buffer sizing is the most impactful TCP tuning you can do.
