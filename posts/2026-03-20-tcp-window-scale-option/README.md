# How to Use the TCP Window Scale Option Effectively

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Window Scaling, Networking, Performance, RFC7323

Description: Understand the TCP Window Scale option, how the scale factor is negotiated during the handshake, and how to ensure it is active for maximum throughput.

## Introduction

The TCP Window Scale option (RFC 7323) extends the receive window field beyond its original 16-bit (64KB) limit. The scale factor is negotiated during the three-way handshake and remains fixed for the connection's lifetime. Understanding how the scale factor is chosen and how to verify it is working helps ensure you're getting maximum throughput from your TCP connections.

## TCP Window Scale Option Format

```text
TCP Option Type: 3 (Window Scale)
Option Length: 3 bytes
Shift Count: 0-14 (the scale factor)

Effective Window = Window Field × 2^(Shift Count)
Scale=0:  64KB × 2^0  = 64KB   (no scaling)
Scale=7:  64KB × 2^7  = 8MB
Scale=10: 64KB × 2^10 = 64MB
Scale=14: 64KB × 2^14 = 1GB    (maximum)
```

## How the Scale Factor is Determined

Linux selects the window scale factor based on the `tcp_rmem` max value:

```bash
# The formula: scale = ceil(log2(tcp_rmem_max / 65536))

# With default tcp_rmem max = 6MB:

# scale = ceil(log2(6291456 / 65536)) = ceil(log2(96)) = ceil(6.58) = 7
# Effective max window = 65535 × 2^7 = 8,388,480 bytes ≈ 8MB

# With tcp_rmem max = 16MB:
# scale = ceil(log2(16777216 / 65536)) = ceil(log2(256)) = 8
# Effective max window = 65535 × 2^8 = 16,776,960 bytes ≈ 16MB

# Check what your system will negotiate
python3 -c "
import math
rmem_max = 16777216  # your tcp_rmem max
scale = math.ceil(math.log2(rmem_max / 65536))
print(f'tcp_rmem max: {rmem_max/1024/1024:.0f} MB')
print(f'Window scale: {scale}')
print(f'Max effective window: {65535 * 2**scale / 1024 / 1024:.1f} MB')
"
```

## Verifying Window Scale Negotiation

```bash
# Capture a TCP handshake and check for window scale option
tcpdump -i eth0 -n -v 'tcp[tcpflags] & tcp-syn != 0' -c 5

# Look for: options [..., nop, wscale 7]
# Or: options [mss 1460,sackOK,TS val 1234 ecr 0,nop,wscale 7]

# If wscale is missing in BOTH SYN and SYN-ACK:
# Window scaling disabled, or one side doesn't support it
```

## Window Scale in the SYN Handshake

```bash
# Wireshark filter to see window scale option
# tcp.flags.syn == 1 and tcp.options.wscale

# In Wireshark packet view:
# Transmission Control Protocol
#   Options: (...)
#     Window scale: 7 (multiply by 128)
#       Kind: Window Scale (3)
#       Length: 3
#       Shift count: 7
```

## Troubleshooting Missing Window Scale

```bash
# Problem: window scaling not being used despite large buffers
# Step 1: Confirm tcp_rmem is large
sysctl net.ipv4.tcp_rmem

# Step 2: Check that tcp_window_scaling is enabled
sysctl net.ipv4.tcp_window_scaling   # Must be 1

# Step 3: Confirm in packet capture
tcpdump -i eth0 -n -v 'tcp[tcpflags] & tcp-syn != 0' | grep wscale

# Step 4: If missing from SYN-ACK but present in SYN:
# Remote server doesn't support window scaling (very old OS or misconfigured)
# Only solution: update the remote OS

# Step 5: Check if any iptables rules modify window size
iptables -L -t mangle -n | grep -i "window"
```

## Conclusion

The TCP Window Scale option is the enabling mechanism for high-throughput TCP. It's automatically negotiated based on your `tcp_rmem` max setting. Ensure window scaling is enabled (sysctl tcp_window_scaling=1), set tcp_rmem max to at least your BDP, and verify in packet captures that the wscale option appears in the SYN handshake. Without it, your connections are limited to 64KB regardless of how large you set your buffers.
