# How to Configure IRQ Affinity for Network Interfaces on Multi-Core Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IRQ, Affinity, Linux, Network Performance, Multi-Core, NUMA

Description: Learn how to configure IRQ affinity for network interface interrupts to distribute packet processing across CPU cores and maximize throughput on multi-core systems.

## Why IRQ Affinity Matters

Every received network packet triggers a hardware interrupt (IRQ) that wakes a CPU to process it. By default, all interrupts for a NIC may land on CPU 0, creating a bottleneck.

Pinning different NIC queue IRQs to different CPU cores:
- Eliminates the single-CPU bottleneck
- Improves cache efficiency (packets stay in the assigned CPU's cache)
- Enables NUMA-aware processing (NIC IRQs on the same NUMA node as the NIC)

## Step 1: Identify NIC IRQs

```bash
# List all IRQs and their associated devices

cat /proc/interrupts | grep eth0

# Output:
# 32:  5234891   0   0   0  PCI-MSI 524288-edge  eth0-TxRx-0
# 33:   987654   0   0   0  PCI-MSI 524289-edge  eth0-TxRx-1
# 34:  4321098   0   0   0  PCI-MSI 524290-edge  eth0-TxRx-2
# 35:  1234567   0   0   0  PCI-MSI 524291-edge  eth0-TxRx-3

# Extract IRQ numbers
IRQ_LIST=$(grep eth0 /proc/interrupts | awk -F: '{print $1}' | tr -d ' ')
echo "eth0 IRQs: $IRQ_LIST"
```

## Step 2: Check Current CPU Affinity

```bash
# Check affinity for each IRQ
for irq in $(grep eth0 /proc/interrupts | awk -F: '{print $1}' | tr -d ' '); do
  MASK=$(cat /proc/irq/$irq/smp_affinity)
  echo "IRQ $irq: smp_affinity = 0x$MASK"
done

# Convert hex mask to CPU list
# 0x1 = CPU 0 only
# 0x2 = CPU 1 only
# 0xff = CPUs 0-7
# 0xf = CPUs 0-3
```

## Step 3: Set IRQ Affinity Manually

```bash
# Pin eth0-TxRx-0 (IRQ 32) to CPU 0
echo 1 > /proc/irq/32/smp_affinity    # CPU 0 = 0x01 = 1

# Pin eth0-TxRx-1 (IRQ 33) to CPU 1
echo 2 > /proc/irq/33/smp_affinity    # CPU 1 = 0x02 = 2

# Pin eth0-TxRx-2 (IRQ 34) to CPU 2
echo 4 > /proc/irq/34/smp_affinity    # CPU 2 = 0x04 = 4

# Pin eth0-TxRx-3 (IRQ 35) to CPU 3
echo 8 > /proc/irq/35/smp_affinity    # CPU 3 = 0x08 = 8
```

## Step 4: Automate with set_irq_affinity Script

```bash
# Script to automatically distribute NIC IRQs across CPUs
cat > /usr/local/sbin/set-irq-affinity.sh << 'SCRIPT'
#!/bin/bash
IFACE=${1:-eth0}
CPUS=$(nproc)

IRQ_LIST=($(grep $IFACE /proc/interrupts | awk -F: '{print $1}' | tr -d ' '))
echo "Found ${#IRQ_LIST[@]} IRQs for $IFACE"

CPU=0
for IRQ in "${IRQ_LIST[@]}"; do
    MASK=$(printf "%x" $((1 << (CPU % CPUS))))
    echo $MASK > /proc/irq/$IRQ/smp_affinity
    echo "IRQ $IRQ -> CPU $CPU (mask 0x$MASK)"
    CPU=$((CPU + 1))
done
SCRIPT

chmod +x /usr/local/sbin/set-irq-affinity.sh
sudo /usr/local/sbin/set-irq-affinity.sh eth0
```

## Step 5: NUMA-Aware IRQ Assignment

For systems with multiple CPU sockets, assign NIC IRQs to the NUMA node closest to the NIC:

```bash
# Find which NUMA node the NIC is on
cat /sys/class/net/eth0/device/numa_node
# Output: 0  (NIC is on NUMA node 0)

# Get CPUs on NUMA node 0
cat /sys/devices/system/node/node0/cpulist
# Output: 0-15,32-47 (CPUs on node 0)

# Assign all NIC IRQs to CPUs on NUMA node 0 only
# This avoids costly cross-NUMA memory accesses
for IRQ in $(grep eth0 /proc/interrupts | awk -F: '{print $1}' | tr -d ' '); do
  # Set affinity to CPUs 0-15 (mask 0xffff)
  echo ffff > /proc/irq/$IRQ/smp_affinity
done
```

## Step 6: Use irqbalance for Automatic Management

For dynamic workloads, `irqbalance` automatically redistributes IRQs:

```bash
# Install irqbalance
sudo apt-get install -y irqbalance

# Configure irqbalance (optional: ban CPUs from IRQ assignment)
cat > /etc/default/irqbalance << 'EOF'
IRQBALANCE_BANNED_CPUS="f000f000"  # Hex mask of CPUs to exclude
IRQBALANCE_ARGS="--deepestcache=2"  # Optimize for L2 cache
EOF

sudo systemctl enable --now irqbalance

# Check irqbalance status
systemctl status irqbalance
```

## Step 7: Verify Distribution

```bash
# Run traffic and watch per-CPU interrupt counts
watch -n 1 "cat /proc/interrupts | grep eth0"

# Use mpstat to verify CPU distribution
mpstat -P ALL 1 10

# Expected: approximately equal %irq across CPUs
# Bad sign: CPU 0 at 90% irq, others near 0%
```

## Conclusion

IRQ affinity configuration routes network interrupts to specific CPU cores, eliminating the single-CPU bottleneck for high-throughput NICs. Distribute queue IRQs evenly across CPUs with the `smp_affinity` files, use NUMA-aware assignment on multi-socket systems, and automate with the provided shell script. For dynamic workloads, `irqbalance` provides automatic redistribution. The result is near-linear scaling of packet processing throughput with CPU count.
