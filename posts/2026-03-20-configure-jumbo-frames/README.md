# How to Configure Jumbo Frames and Verify MTU Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, Jumbo Frames, Networking, Linux, Performance, Ethernet

Description: Enable and verify jumbo frame support (9000 MTU) on Linux servers and network interfaces for improved throughput in high-performance LAN environments.

## Introduction

Jumbo frames are Ethernet frames with a payload larger than the standard 1500-byte MTU. Standard jumbo frame size is 9000 bytes (9000 MTU), though the exact maximum varies by hardware. Enabling jumbo frames on a LAN reduces CPU overhead for large transfers by requiring fewer packets to transmit the same data. All devices on the path must support jumbo frames for this to work.

## Check Current MTU

```bash
# Check current MTU of all interfaces:

ip link show | grep mtu

# Check if NIC supports jumbo frames:
ethtool -k eth0 | grep -i large
# Or check maximum supported MTU:
ip link set eth0 mtu 9001 2>&1
# If error "SIOCSIFMTU: Invalid argument": NIC doesn't support this MTU
# If success: MTU is now 9001
ip link set eth0 mtu 1500  # Restore to standard
```

## Enable Jumbo Frames on Linux

```bash
# Temporarily set MTU to 9000 (jumbo frames):
ip link set eth0 mtu 9000

# Verify the change:
ip link show eth0 | grep mtu
# Should show: mtu 9000

# Test jumbo frames work:
ping -M do -s 8972 -c 5 10.20.0.2  # Test to another host on same LAN
# 9000 - 28 (ICMP headers) = 8972 bytes payload

# If ping fails: switch port or remote host doesn't support jumbo frames
```

## Make Jumbo Frames Persistent

```bash
# Method 1: systemd-networkd:
cat > /etc/systemd/network/10-eth0.network << 'EOF'
[Match]
Name=eth0

[Link]
MTUBytes=9000
EOF
networkctl reload

# Method 2: NetworkManager:
nmcli connection modify "Wired connection 1" 802-3-ethernet.mtu 9000
nmcli connection up "Wired connection 1"

# Method 3: Netplan (Ubuntu):
cat > /etc/netplan/01-eth0.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      mtu: 9000
EOF
netplan apply

# Method 4: /etc/network/interfaces (Debian):
cat >> /etc/network/interfaces << 'EOF'
iface eth0 inet dhcp
    mtu 9000
EOF
```

## Verify All Hosts in Path Support Jumbo Frames

```bash
#!/bin/bash
# Test jumbo frame support on multiple hosts

HOSTS=("10.20.0.10" "10.20.0.11" "10.20.0.12")
JUMBO_SIZE=8972  # 9000 - 28 bytes headers

echo "Jumbo Frame Support Test:"
echo "========================="

for host in "${HOSTS[@]}"; do
    if ping -M do -s $JUMBO_SIZE -c 3 -W 3 $host > /dev/null 2>&1; then
        echo "  $host: SUPPORTED (9000 MTU)"
    else
        # Find what size works:
        if ping -M do -s 1472 -c 1 -W 2 $host > /dev/null 2>&1; then
            echo "  $host: STANDARD MTU ONLY (1500)"
        else
            echo "  $host: UNREACHABLE or MTU < 1500"
        fi
    fi
done

# Also test the switch/path:
echo ""
echo "Path MTU to each host:"
for host in "${HOSTS[@]}"; do
    PMTU=$(tracepath -n $host 2>/dev/null | grep pmtu | tail -1 | grep -oP 'pmtu \K[0-9]+')
    echo "  $host: ${PMTU:-unknown} bytes"
done
```

## Performance Comparison

```bash
# Compare throughput with standard vs jumbo frames:

# Test with standard MTU (1500):
ip link set eth0 mtu 1500
iperf3 -c 10.20.0.5 -t 30 -P 4 2>&1 | grep receiver

# Test with jumbo frames (9000):
ip link set eth0 mtu 9000
iperf3 -c 10.20.0.5 -t 30 -P 4 2>&1 | grep receiver

# Typical improvement: 5-10% higher throughput
# More significant reduction: CPU usage (fewer interrupts per byte transferred)

# Check CPU during transfer:
vmstat 1 5 | awk '{print "usr:", $13, "sys:", $14}'
# Lower "sys" value with jumbo frames = less kernel overhead
```

## Switch/Network Requirements

```text
For jumbo frames to work:
1. All NIC adapters on the path: must support 9000 MTU
2. All switch ports: must be configured for jumbo frames
   (Cisco: mtu 9216 on interface; Arista: mtu 9214)
3. All virtual switches: VMware, OVS, Linux bridge must be configured
4. All routers: frame size must accommodate jumbo packets
   (Routers between subnets may need configuration)

Partial jumbo support = frames get fragmented or dropped at the non-jumbo hop
```

## Conclusion

Jumbo frames improve large-data LAN performance by reducing packet overhead and CPU interrupts. Enable with `ip link set eth0 mtu 9000` and test with `ping -M do -s 8972`. All devices on the LAN path (NICs, switches, VMs) must support the jumbo frame size. Verify with the test script before enabling in production. The performance gain is most noticeable for storage replication, backup streams, and HPC workloads - typical improvement is 5-10% throughput and a significant reduction in CPU utilization for network-intensive workloads.
