# How to Enable and Configure TCP BBR on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, BBR, Linux, Congestion Control, Performance, Networking

Description: Enable TCP BBR congestion control on Linux and configure it optimally for high-bandwidth, high-latency, or lossy network links.

## Introduction

BBR (Bottleneck Bandwidth and RTT) is Google's TCP congestion control algorithm, available in Linux kernel 4.9+. Unlike loss-based algorithms (CUBIC, Reno), BBR estimates the true available bandwidth and minimum path RTT to set its sending rate. This makes it dramatically more effective on long-distance or lossy links where packet loss occurs even without congestion.

## Prerequisites and Installation

```bash
# Check kernel version (BBR requires 4.9+)

uname -r
# Should show 4.9 or higher

# Load the BBR module
modprobe tcp_bbr

# Verify BBR is available
sysctl net.ipv4.tcp_available_congestion_control | grep bbr

# If BBR not listed: kernel is too old, or module not compiled
# Solution: upgrade kernel or compile with CONFIG_TCP_CONG_BBR=m
```

## Enabling BBR

```bash
# Enable BBR as the default congestion control
sysctl -w net.ipv4.tcp_congestion_control=bbr

# BBR requires the fq (Fair Queue) pacing scheduler for best results
sysctl -w net.core.default_qdisc=fq

# Verify both settings
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = bbr

sysctl net.core.default_qdisc
# net.core.default_qdisc = fq

# Make permanent
cat >> /etc/sysctl.conf << EOF
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sysctl -p

# Auto-load module on boot
echo "tcp_bbr" >> /etc/modules
```

## Why BBR Needs the fq Qdisc

BBR controls sending rate through pacing - sending packets at a calculated rate rather than bursting them all at once. The `fq` (Fair Queue) qdisc implements per-flow pacing at the kernel level:

```bash
# Check current qdisc on each interface
tc qdisc show dev eth0

# Apply fq qdisc manually if needed
tc qdisc replace dev eth0 root fq

# The sysctl net.core.default_qdisc=fq applies to all new interfaces
# Existing interfaces may need manual tc qdisc replace
```

## Verifying BBR is Active

```bash
# Confirm BBR is running on active connections
ss -tin state established | grep "bbr"
# Look for: "cc:bbr" in the output

# More detailed BBR statistics
ss -tin state established | head -5
# Output includes:
# cubic, reno, or bbr in the cc: field
# pacing_rate: shows BBR's current pacing rate

# Monitor BBR congestion window during a transfer
watch -n 0.5 'ss -tin state established | grep -E "cc:|cwnd|pacing"'
```

## BBR Performance Testing

```bash
# Baseline with CUBIC
sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c 10.20.0.5 -t 30
echo "CUBIC result above"

# Switch to BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr
iperf3 -c 10.20.0.5 -t 30
echo "BBR result above"

# For high-latency links (simulate with tc netem):
tc qdisc add dev eth0 root netem delay 100ms loss 1%
iperf3 -c 10.20.0.5 -t 30   # Much better with BBR on lossy/latency path
tc qdisc del dev eth0 root
```

## When BBR Excels vs When to Stick with CUBIC

```text
Use BBR for:
- WAN links with RTT > 50ms
- Satellite links (300-600ms RTT)
- Networks with 0.5-2% background loss
- Long-distance bulk transfers (data center to cloud, etc.)

Keep CUBIC for:
- Pure LAN (<5ms RTT, near-zero loss)
- Legacy systems that may not play well with BBR's behavior
- Environments where single-flow fairness is critical
```

## Conclusion

BBR is now the recommended congestion control for most internet-facing services. Enabling it takes two sysctl settings: `tcp_congestion_control=bbr` and `default_qdisc=fq`. The performance improvement is most dramatic on high-latency or lossy paths - expect 3-10× throughput improvement compared to CUBIC on satellite or transcontinental links.
