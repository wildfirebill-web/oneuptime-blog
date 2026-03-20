# How to Optimize Linux Network Stack with sysctl for 10Gbps Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Sysctl, Performance, 10Gbps, Network Tuning, TCP

Description: Learn the complete set of sysctl parameters to optimize the Linux network stack for 10Gbps network adapters, covering buffer sizes, TCP behavior, and memory limits.

## Why Default Linux Settings Are Insufficient for 10G

Linux ships with conservative defaults tuned for small servers with limited RAM. On a 10 Gbps network with 10ms RTT, the theoretical maximum per-connection throughput requires:

```text
BDP = 10 Gbps × 0.010s = 100,000,000 bits = 12.5 MB per connection
```

Default TCP buffers (~200 KB) cap throughput at ~160 Mbps on such a link - far below 10 Gbps capability.

## Step 1: Check Current Baseline

```bash
# View key current settings

sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
sysctl net.core.netdev_max_backlog
sysctl net.ipv4.tcp_congestion_control

# Test current throughput
iperf3 -c <server-ip> -t 30 -P 4
```

## Step 2: Apply the 10G Tuning Profile

Create a comprehensive sysctl configuration:

```bash
cat > /etc/sysctl.d/99-10g-tuning.conf << 'EOF'
###############################################
# Socket Buffer Sizes
###############################################
# Maximum socket receive/send buffer (256 MB)
net.core.rmem_max = 268435456
net.core.wmem_max = 268435456

# Default socket buffer sizes (for non-TCP sockets like UDP)
net.core.rmem_default = 33554432
net.core.wmem_default = 33554432

# TCP buffer sizes: min/default/max
# Auto-tuning grows within this range per connection
net.ipv4.tcp_rmem = 4096 1048576 268435456
net.ipv4.tcp_wmem = 4096 1048576 268435456

###############################################
# TCP Memory
###############################################
# System-wide TCP memory limits (in 4K pages)
# min/pressure/max
# For 64GB system: ~1GB min, 4GB pressure, 8GB max
net.ipv4.tcp_mem = 262144 1048576 2097152

###############################################
# Receive Queue
###############################################
# Increase netdev receive queue for burst traffic
net.core.netdev_max_backlog = 30000

# Enable busy polling for lowest latency
net.core.busy_poll = 50
net.core.busy_read = 50

###############################################
# TCP Behavior
###############################################
# Enable auto-tuning (default on, but confirm)
net.ipv4.tcp_moderate_rcvbuf = 1

# Enable BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Enable SACK
net.ipv4.tcp_sack = 1
net.ipv4.tcp_dsack = 1

# Window scaling (required for large windows)
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_timestamps = 1

###############################################
# Connection Handling
###############################################
# Listen backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# SYN cookies
net.ipv4.tcp_syncookies = 1

# Reduce TIME_WAIT duration
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

# Local port range for outbound connections
net.ipv4.ip_local_port_range = 1024 65535
EOF

# Apply all settings
sudo sysctl -p /etc/sysctl.d/99-10g-tuning.conf
```

## Step 3: Enable Network Interface Offloads

Offloads reduce CPU load for 10G processing:

```bash
# Replace eth0 with your actual interface name
IFACE=eth0

# Enable TX checksum offload
ethtool -K $IFACE tx-checksum-ipv4 on tx-checksum-ipv6 on

# Enable TCP Segmentation Offload (TSO)
ethtool -K $IFACE tso on

# Enable Generic Receive Offload (GRO)
ethtool -K $IFACE gro on

# Enable Large Receive Offload (LRO) - if supported
ethtool -K $IFACE lro on

# Verify offloads are enabled
ethtool -k $IFACE | grep -E "tcp-seg|generic-receive|large-receive|checksum"
```

## Step 4: Increase Ring Buffer Sizes

```bash
# Check current ring buffer sizes
ethtool -g eth0

# Set to maximum for 10G
ethtool -G eth0 rx 4096 tx 4096

# Persist ring buffer settings across reboots
# Create /etc/udev/rules.d/99-net-ring.rules
cat > /etc/udev/rules.d/99-net-ring.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", RUN+="/sbin/ethtool -G eth0 rx 4096 tx 4096"
EOF
```

## Step 5: Verify Throughput Improvement

```bash
# Test with iperf3 - multiple streams
# Server side (remote):
iperf3 -s -p 5201

# Client side - 4 parallel streams, 60 second test
iperf3 -c server-ip -t 60 -P 4 -p 5201

# Expected output after tuning on 10G LAN:
# [SUM] 0.00-60.00 sec  xx.x GBytes  xx.x Gbits/sec   sender
# [SUM] 0.00-60.00 sec  xx.x GBytes  xx.x Gbits/sec   receiver

# Test reverse (server sends to client)
iperf3 -c server-ip -t 60 -P 4 -R
```

## Conclusion

Tuning a Linux system for 10 Gbps requires increasing socket buffer sizes to match the Bandwidth-Delay Product, enabling offloads like GRO and TSO, increasing ring buffers, and switching to BBR congestion control. Apply the settings in `/etc/sysctl.d/99-10g-tuning.conf` for persistence, and verify with iperf3 multi-stream tests. The combination of large buffers and BBR typically achieves 90%+ of theoretical 10 Gbps throughput.
