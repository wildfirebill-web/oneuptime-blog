# How to Use ethtool to Tune Network Interface Ring Buffers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ethtool, Ring Buffer, Linux, Network Performance, NIC Tuning

Description: Learn how to use ethtool to view and increase NIC ring buffer sizes, preventing packet drops under high traffic bursts on Linux servers.

## What Are Ring Buffers?

A NIC ring buffer (also called a descriptor ring) is a circular queue between the NIC hardware and the kernel. Incoming packets are placed in the receive ring; outgoing packets are placed in the transmit ring.

If the ring fills up because the CPU can't drain it fast enough, packets are dropped at the NIC level - before they ever reach the kernel TCP stack. This is a common cause of packet loss on high-traffic servers.

## Step 1: Check Current Ring Buffer Sizes

```bash
# View current and maximum ring buffer sizes

ethtool -g eth0

# Output:
# Ring parameters for eth0:
# Pre-set maximums:
# RX:          4096     <- hardware maximum for receive ring
# RX Mini:     0
# RX Jumbo:    0
# TX:          4096     <- hardware maximum for transmit ring
#
# Current hardware settings:
# RX:          256      <- current receive ring size (often default)
# RX Mini:     0
# RX Jumbo:    0
# TX:          256      <- current transmit ring size

# If Current < Pre-set maximums, you can increase it
```

## Step 2: Check for Ring Buffer Drops

```bash
# Check if ring buffer drops are occurring
ethtool -S eth0 | grep -i "drop\|missed\|rx_no_buffer\|rx_fifo"

# Common drop counters:
# rx_dropped: packets dropped due to full receive ring
# rx_missed_errors: frames missed
# tx_dropped: packets dropped due to full transmit ring

# Also check with netstat
netstat -i
# RX-DRP and TX-DRP columns show drops per interface
```

## Step 3: Increase Ring Buffer Sizes

```bash
# Increase RX and TX ring to maximum supported size
ethtool -G eth0 rx 4096 tx 4096

# Verify change applied
ethtool -g eth0 | grep "Current hardware" -A5
```

## Step 4: Monitor Ring Buffer Utilization

```bash
# Watch drops in real time while generating traffic
watch -n 1 "ethtool -S eth0 | grep -i drop"

# Use sar to track packet drops over time
sar -n EDEV 1 60

# Output:
# Average:     IFACE    rxerr/s  txerr/s  coll/s  rxdrop/s  txdrop/s
# Average:      eth0       0.00     0.00    0.00     5.23      0.00
# rxdrop/s > 0 means ring buffer is overflowing
```

## Step 5: Tune Ring Buffer with Traffic Shaping

Large ring buffers reduce drops but increase latency (more buffering = more queuing delay). For latency-sensitive workloads:

```bash
# Balance ring size vs latency
# For latency-sensitive (gaming, HFT, real-time control):
ethtool -G eth0 rx 256 tx 256

# For throughput-optimized (file transfer, backup, media):
ethtool -G eth0 rx 4096 tx 4096

# For mixed workloads:
ethtool -G eth0 rx 1024 tx 1024
```

## Step 6: Make Ring Buffer Settings Persistent

```bash
# Method 1: udev rule
cat > /etc/udev/rules.d/99-ring-buffer.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", \
  RUN+="/sbin/ethtool -G eth0 rx 4096 tx 4096"
EOF

# Method 2: Systemd service
cat > /etc/systemd/system/ring-buffer-tuning.service << 'EOF'
[Unit]
Description=Set NIC ring buffer sizes
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -G eth0 rx 4096 tx 4096
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable ring-buffer-tuning

# Method 3: For multiple interfaces
for iface in eth0 eth1; do
  ethtool -G $iface rx 4096 tx 4096 2>/dev/null && echo "$iface: ring buffer set"
done
```

## Step 7: Check Other ethtool Tuning Options

```bash
# Check and set coalescing (interrupt aggregation)
ethtool -c eth0

# Increase coalescing to reduce interrupt rate (improves throughput at cost of latency)
ethtool -C eth0 rx-usecs 50 tx-usecs 50

# Check driver information
ethtool -i eth0

# Check link speed and duplex
ethtool eth0 | grep -E "Speed|Duplex|Auto"
```

## Conclusion

NIC ring buffer drops are a silent source of packet loss on busy servers. Check for drops with `ethtool -S eth0 | grep drop`, increase ring size to hardware maximum with `ethtool -G eth0 rx 4096 tx 4096`, and monitor with `sar -n EDEV`. Persist settings in udev rules or a systemd service. For latency-sensitive workloads, balance ring size with coalescing settings to avoid excessive queuing delay.
