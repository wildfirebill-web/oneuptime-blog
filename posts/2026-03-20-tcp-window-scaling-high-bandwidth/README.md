# How to Enable TCP Window Scaling for High-Bandwidth Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Window Scaling, Performance, High Bandwidth, Networking

Description: Enable and verify TCP window scaling to overcome the 64KB receive window limitation for high-bandwidth or high-latency network connections.

## Introduction

The original TCP specification limits the receive window field to 16 bits - a maximum of 65,535 bytes (64KB). For modern networks with gigabit bandwidth and milliseconds of latency, this small window severely limits throughput. TCP Window Scaling (RFC 7323) extends the window using a scale factor negotiated during the handshake, allowing windows up to 1 gigabyte.

## How Window Scaling Works

```text
During handshake:
Client SYN includes: TCP option "window scale = 7"
Server SYN-ACK includes: TCP option "window scale = 10"

During data transfer:
Client advertises window = 65535 in header
Effective window = 65535 × 2^7 = 8,388,480 bytes ≈ 8MB

Server advertises window = 65535 in header
Effective window = 65535 × 2^10 = 67,107,840 bytes ≈ 64MB
```

## Verifying Window Scaling is Enabled

```bash
# Check kernel setting (should be 1 by default on modern Linux)

sysctl net.ipv4.tcp_window_scaling
# net.ipv4.tcp_window_scaling = 1

# Verify in a packet capture: look for wscale option in SYN packets
tcpdump -i eth0 -n -v 'tcp[tcpflags] & tcp-syn != 0 and port 80'
# Look for: options [..., wscale 7, nop,nop]

# If wscale is missing: window scaling not negotiated
# Cause: older OS/device on path, or misconfiguration
```

## Enabling Window Scaling

```bash
# Enable window scaling (should already be on in Linux 2.6+)
sysctl -w net.ipv4.tcp_window_scaling=1

# Ensure max buffer sizes are large enough to take advantage
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 87380 16777216"
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216

# Persist
echo "net.ipv4.tcp_window_scaling=1" >> /etc/sysctl.conf
```

## Diagnosing Window Scaling Issues

```bash
# If window scaling is negotiated but effective window seems small:
# Check what scale factor was negotiated
tcpdump -i eth0 -n -v 'tcp[tcpflags] & tcp-syn != 0' -c 5 2>/dev/null

# If wscale is 0 or 1: scale factor is very small
# Desired: wscale 7 or higher (128× or more)

# What determines the scale factor?
# Linux selects scale factor based on tcp_rmem max value
# With max=6MB: scale = 7 (128×, giving max window = 64KB × 128 = 8MB)
# With max=16MB: scale = 8 (256×, giving max window = 64KB × 256 = 16MB)
```

## Verifying Effective Window in a Transfer

```bash
# Start a large transfer
scp large_file.tar.gz user@10.20.0.5:/tmp/

# In another terminal, monitor the window sizes being advertised
tcpdump -i eth0 -n -v 'tcp and host 10.20.0.5 and port 22' 2>/dev/null | \
  awk '/win /{match($0, /win ([0-9]+)/, a); if(a[1]+0>10000) print a[1]" bytes"}'

# Values much larger than 65535 confirm window scaling is active
```

## Middle Box Issues with Window Scaling

```bash
# Some old firewalls/NAT devices strip or modify window scaling options
# Symptom: connection works but throughput is limited to ~5 Mbps (64KB/10ms RTT)

# Test: compare throughput with window scaling vs without
sysctl -w net.ipv4.tcp_window_scaling=0   # Disable temporarily
iperf3 -c 10.20.0.5 -t 10
sysctl -w net.ipv4.tcp_window_scaling=1   # Re-enable
iperf3 -c 10.20.0.5 -t 10

# If no difference in throughput: middlebox stripping scaling options
# Fix: update/replace the problematic middlebox
```

## Conclusion

TCP window scaling is enabled by default on all modern Linux systems and is essential for high-throughput networking. The scale factor is automatically negotiated based on your buffer size settings. To maximize throughput on high-latency links, ensure your `tcp_rmem` max is set to at least the bandwidth-delay product for your network, and verify the scale factor in packet captures to confirm scaling is actually active.
