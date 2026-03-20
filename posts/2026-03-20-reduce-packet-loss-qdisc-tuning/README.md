# How to Reduce IPv4 Packet Loss with Queue Discipline (qdisc) Tuning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Qdisc, tc, Traffic Control, Linux, Packet Loss, QoS, FQ-CoDel

Description: Learn how to configure Linux traffic control queue disciplines (qdiscs) to reduce packet loss, latency, and bufferbloat on network interfaces.

## What Is a Queue Discipline?

Every network interface on Linux has a queue discipline (qdisc) that controls how packets are queued and transmitted. The default qdisc (`pfifo_fast`) uses simple three-band priority queuing with a fixed queue length.

Problems with default qdisc:
- Fixed queue depth can cause bursts of packet loss
- No active queue management (AQM) leads to bufferbloat
- No fairness between flows

## Step 1: Check Current qdisc Configuration

```bash
# View current qdisc for all interfaces

tc qdisc show

# Output:
# qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
# qdisc pfifo_fast 0: dev lo root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1

# Check queue statistics (drops, backlog)
tc -s qdisc show dev eth0
# Output includes: Sent, Dropped, Overlimits, Backlog
```

## Step 2: Switch to FQ-CoDel (Best General Purpose)

FQ-CoDel (Fair Queue Controlled Delay) is the recommended qdisc for most systems. It:
- Provides fair bandwidth between flows
- Actively manages queue depth (CoDel AQM)
- Drastically reduces bufferbloat
- Is included in Linux kernel 3.5+

```bash
# Replace default pfifo_fast with fq_codel
sudo tc qdisc replace dev eth0 root fq_codel

# Verify
tc qdisc show dev eth0
# qdisc fq_codel 8001: root refcnt 2 limit 10240 flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64

# Check statistics after running some traffic
tc -s qdisc show dev eth0
```

## Step 3: Configure CAKE (Modern Alternative to FQ-CoDel)

CAKE (Common Applications Kept Enhanced) is a newer AQM that also handles shaping:

```bash
# Install CAKE (if not already in kernel)
# CAKE is included in kernel 4.19+
modprobe sch_cake

# Apply CAKE with bandwidth limiting (replace 100mbit with your uplink speed)
sudo tc qdisc replace dev eth0 root cake bandwidth 100mbit

# CAKE configuration options:
sudo tc qdisc replace dev eth0 root cake \
  bandwidth 100mbit \    # Your uplink bandwidth
  besteffort \           # No classification (use for most cases)
  ack-filter \           # Filter redundant ACKs
  nat                    # Handle NAT for RTT estimation

# Verify
tc qdisc show dev eth0
```

## Step 4: Increase pfifo_fast Queue Length (Simple Fix)

If you need a quick fix without changing the qdisc type:

```bash
# Check current queue length
ip link show eth0 | grep qlen

# Increase queue length (txqueuelen)
sudo ip link set eth0 txqueuelen 10000

# Verify
ip link show eth0 | grep qlen
# ... qlen 10000 ...

# Make persistent
echo 'ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", ATTR{tx_queue_len}="10000"' \
  > /etc/udev/rules.d/99-txqueuelen.rules
```

## Step 5: Configure Per-Flow Fairness with FQ

The `fq` qdisc provides per-flow fairness without active queue management (used with BBR):

```bash
# fq is required when using BBR congestion control
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# Or apply manually to an interface
sudo tc qdisc replace dev eth0 root fq maxrate 0 quantum 1500

# fq automatically creates per-flow queues and provides pacing
tc qdisc show dev eth0
```

## Step 6: Diagnose Packet Drops with tc Statistics

```bash
# Monitor qdisc drops in real time
watch -n 1 "tc -s qdisc show dev eth0 | grep -E 'Sent|drop|backlog'"

# Output:
# Sent 123456789 bytes 98765 pkt (dropped 0, overlimits 0 requeues 5)
# backlog 0b 0p requeues 5

# High "dropped" count = qdisc is dropping due to queue overflow
# High "backlog" count = queue is filling up (risk of drops and latency)

# Compare with NIC-level drops
ethtool -S eth0 | grep -i drop
```

## Step 7: Set Default qdisc System-Wide

```bash
# Set default qdisc for all new interfaces
echo "net.core.default_qdisc = fq_codel" >> /etc/sysctl.d/99-qdisc.conf
sudo sysctl -p /etc/sysctl.d/99-qdisc.conf

# Apply fq_codel to all existing interfaces
for iface in $(ip link show | grep '^[0-9]' | awk -F: '{print $2}' | tr -d ' ' | grep -v lo); do
  tc qdisc replace dev $iface root fq_codel
  echo "Applied fq_codel to $iface"
done
```

## Conclusion

The Linux qdisc controls packet queuing behavior and is a key factor in packet loss and latency. Replace the default `pfifo_fast` with `fq_codel` for better bufferbloat handling and fair bandwidth distribution, or use `fq` alongside BBR congestion control. Set the system-wide default with `net.core.default_qdisc = fq_codel` in sysctl and monitor drop counts with `tc -s qdisc show`. For bandwidth-limited uplinks, CAKE provides combined shaping and AQM in a single qdisc.
