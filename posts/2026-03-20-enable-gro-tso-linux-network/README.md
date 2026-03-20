# How to Enable Generic Receive Offload (GRO) and TCP Segmentation Offload (TSO)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GRO, TSO, Linux, Network Performance, ethtool, Offload

Description: Learn how to enable GRO and TSO network offloads on Linux to reduce CPU overhead for high-throughput network workloads.

## What Are GRO and TSO?

Modern NICs can offload CPU-intensive packet processing to hardware:

- **TSO (TCP Segmentation Offload)** - The kernel sends large chunks of data to the NIC, which splits them into MTU-sized segments. Without TSO, the kernel CPU does the splitting.

- **GRO (Generic Receive Offload)** - The NIC (or kernel) combines multiple small incoming TCP segments into one large buffer before passing to the TCP stack. This reduces per-packet overhead for receives.

These offloads can double or triple throughput on busy servers by reducing CPU cycles spent on packet processing.

## Step 1: Check Current Offload Status

```bash
# View all offload settings for an interface

ethtool -k eth0

# Key offloads to check:
# tx-checksumming: on
# scatter-gather: on
# tcp-segmentation-offload: on
# generic-receive-offload: on
# large-receive-offload: on [fixed]
# rx-vlan-offload: on

# Quick check for the important ones
ethtool -k eth0 | grep -E "tcp-seg|generic-rec|large-rec|scatter"
```

## Step 2: Enable TSO

TSO reduces CPU usage for large TCP sends:

```bash
# Enable TSO
ethtool -K eth0 tso on

# Enable related checksum offloads (required for TSO)
ethtool -K eth0 tx on
ethtool -K eth0 sg on    # Scatter-gather (required for TSO)

# Verify
ethtool -k eth0 | grep "tcp-segmentation-offload"
# tcp-segmentation-offload: on
```

## Step 3: Enable GRO

GRO reduces CPU usage for large TCP receives:

```bash
# Enable GRO
ethtool -K eth0 gro on

# Some NICs support LRO (Large Receive Offload - hardware GRO)
# LRO is more efficient but can cause issues with routing
ethtool -K eth0 lro on 2>/dev/null && echo "LRO enabled" || echo "LRO not supported"

# Verify GRO
ethtool -k eth0 | grep "generic-receive-offload"
# generic-receive-offload: on
```

## Step 4: Enable Full Offload Suite

```bash
# Enable comprehensive offload suite for high-performance
IFACE=eth0

ethtool -K $IFACE \
  tso on \
  gso on \
  gro on \
  tx on \
  rx on \
  sg on

# Check result
ethtool -k $IFACE | grep -E "on|off" | head -20
```

## Step 5: Verify Impact with ethtool Statistics

```bash
# Check NIC statistics before and after enabling offloads
ethtool -S eth0 | grep -E "tx_tso|rx_gro|lro"

# Or use /proc/net/dev for overall stats
cat /proc/net/dev | grep eth0

# Generate traffic and measure CPU usage
iperf3 -s &
# On another terminal:
iperf3 -c localhost -t 30 &
mpstat 1 30 | awk '/Average/ {print "CPU idle:", $NF "%"}'
```

## Step 6: TSO and GRO Interaction with Virtualization

In VMs, TSO and GRO may interact with the hypervisor's virtual NIC:

```bash
# For VMware/KVM VMs, check virtio-net offloads
ethtool -k eth0 | grep -E "segmentation|offload"

# If running as a router/bridge, disable LRO to avoid routing issues
# LRO combines packets before routing decisions, causing issues
ethtool -K eth0 lro off

# For DPDK or XDP workloads, all offloads should be checked for compatibility
```

## Step 7: Make Offloads Persistent

```bash
# Method 1: udev rules
cat > /etc/udev/rules.d/99-net-offload.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", \
  RUN+="/sbin/ethtool -K eth0 tso on gso on gro on sg on"
EOF

# Method 2: NetworkManager dispatcher
cat > /etc/NetworkManager/dispatcher.d/99-offloads << 'EOF'
#!/bin/bash
IFACE=$1
EVENT=$2
if [ "$EVENT" = "up" ] && [ "$IFACE" = "eth0" ]; then
    ethtool -K $IFACE tso on gso on gro on
fi
EOF
chmod +x /etc/NetworkManager/dispatcher.d/99-offloads
```

## Conclusion

TSO and GRO are essential offloads for high-throughput Linux servers. TSO lets the NIC handle TCP segmentation instead of the CPU, and GRO batches incoming segments for more efficient processing. Enable both with `ethtool -K eth0 tso on gro on`. On VMs, verify the virtual NIC driver supports these offloads. These settings typically reduce CPU usage by 30-50% for network-intensive workloads while improving throughput.
