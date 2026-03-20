# How to Tune Linux Kernel UDP Parameters for High Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Linux, Kernel, Performance, Sysctl, Tuning

Description: Tune Linux kernel parameters for high-volume UDP traffic including socket buffers, NIC queues, interrupt handling, and memory settings to achieve maximum UDP throughput.

## Introduction

Default Linux kernel settings are conservative and optimized for general-purpose use, not high-volume UDP. A server handling 1 million UDP packets per second needs kernel tuning across multiple subsystems: socket buffers, NIC receive rings, interrupt coalescing, CPU affinity, and memory limits. This guide covers the key parameters for high-traffic UDP applications like DNS resolvers, game servers, and video streaming.

## Socket Buffer Limits

```bash
# Default limits are 212 KB - too small for high-throughput UDP:

sysctl net.core.rmem_max          # Max receive buffer (application can request up to this)
sysctl net.core.wmem_max          # Max send buffer
sysctl net.core.rmem_default      # Default receive buffer (before application sets SO_RCVBUF)
sysctl net.core.wmem_default      # Default send buffer

# For high-throughput:
cat >> /etc/sysctl.conf << 'EOF'
# Socket buffer limits
net.core.rmem_max = 134217728      # 128 MB max receive
net.core.wmem_max = 134217728      # 128 MB max send
net.core.rmem_default = 8388608    # 8 MB default receive
net.core.wmem_default = 8388608    # 8 MB default send
EOF
sysctl -p
```

## NIC Receive Queue Tuning

```bash
# Increase NIC ring buffer size (packets queued at NIC before DMA):
ethtool -g eth0  # Show current ring buffer sizes
ethtool -G eth0 rx 4096 tx 4096  # Increase to 4096 (hardware max varies)

# Increase kernel backlog queue:
sysctl -w net.core.netdev_max_backlog=50000
# Packets queued per CPU between NIC and socket layer

# Allow multiple receive queues (RSS - Receive Side Scaling):
ethtool -l eth0  # Show current queue count
ethtool -L eth0 combined $(nproc)  # Set queues equal to CPU count
# Spreads UDP receive load across all CPU cores
```

## Interrupt Handling (IRQ Affinity)

```bash
# Spread NIC interrupts across CPU cores for parallel processing:

# Check current IRQ distribution:
cat /proc/interrupts | grep eth0

# Set IRQ affinity (each NIC queue to a different CPU):
# Get IRQ numbers for eth0:
ls /sys/class/net/eth0/device/msi_irqs/ 2>/dev/null || \
  grep eth0 /proc/interrupts | awk '{print $1}' | tr -d ':'

# Set CPU affinity for IRQ 42 to CPU 2:
echo 4 > /proc/irq/42/smp_affinity  # CPU 2 (bitmask: 2^2 = 4)

# Automate with irqbalance:
systemctl stop irqbalance  # Stop auto-balancer first
# Then manually assign IRQs, or let irqbalance run:
systemctl start irqbalance
```

## SO_REUSEPORT for UDP Scale-Out

```bash
# Multiple processes can bind to the same UDP port with SO_REUSEPORT
# Kernel load-balances incoming packets across all listeners

# Enable in application (Python):
# sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)

# Start N workers, each binding with SO_REUSEPORT:
# Worker 1: bind('0.0.0.0', 5000) with SO_REUSEPORT
# Worker 2: bind('0.0.0.0', 5000) with SO_REUSEPORT
# ... N workers
# Kernel distributes UDP packets evenly using 4-tuple hash

# This effectively multiplies UDP receive capacity by N
# Critical for DNS resolvers and game servers handling >100K pps
```

## Busy Polling

```bash
# For ultra-low latency UDP: use busy polling instead of interrupts
# CPU spins waiting for packets instead of sleeping and being interrupted
# Eliminates interrupt latency (~10-50µs) at cost of 100% CPU on that core

# Enable busy polling:
sysctl -w net.core.busy_poll=50       # Poll for 50 microseconds
sysctl -w net.core.busy_read=50       # Read busy poll timeout

# Per-socket busy polling (lower overhead):
# sock.setsockopt(socket.SOL_SOCKET, socket.SO_BUSY_POLL, 50)

# Only use for latency-critical applications on dedicated servers
```

## Complete High-Traffic UDP Sysctl Profile

```bash
cat > /etc/sysctl.d/99-udp-highperf.conf << 'EOF'
# Socket buffers
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608

# NIC queue depth
net.core.netdev_max_backlog = 50000
net.core.netdev_budget = 600        # More packets per NAPI poll cycle

# UDP-specific
net.ipv4.udp_mem = 102400 873800 16777216  # min/pressure/max pages for UDP

# Optional: busy polling for lowest latency
# net.core.busy_poll = 50
# net.core.busy_read = 50
EOF
sysctl -p /etc/sysctl.d/99-udp-highperf.conf
```

## Conclusion

High-traffic UDP tuning addresses four areas: socket buffer size (`rmem_max`/`wmem_max`), NIC queue depth (`netdev_max_backlog` and ring buffers), interrupt distribution (IRQ affinity and RSS), and application architecture (`SO_REUSEPORT` for multi-process parallelism). For sub-millisecond latency with millions of packets per second, add busy polling. Apply these settings progressively, measuring packet drop rates (`nstat UdpRcvbufErrors`) at each step to confirm improvement.
