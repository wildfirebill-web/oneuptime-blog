# How to Configure Balance-TLB Bonding (Mode 5) on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bonding, Balance-TLB, Mode 5, Adaptive Load Balancing, Networking

Description: Configure Linux balance-TLB bonding (mode 5) for adaptive transmit load balancing that distributes outgoing traffic based on current load without switch configuration.

## Introduction

Balance-TLB (Adaptive Transmit Load Balancing, mode 5) distributes outgoing traffic across slaves based on current load without requiring switch configuration. Incoming traffic arrives only on the current active slave. If a slave fails, another slave takes over, and the MAC address of the failed slave is reassigned to the next active slave.

## How Balance-TLB Differs from Other Modes

- **No switch configuration required** (unlike modes 0, 2, 4)
- **Transmit**: load balanced across all slaves based on load
- **Receive**: only on the active receive slave
- Requires the `ethtool` utility for load tracking

## Configure Balance-TLB

```bash
# Load bonding module
modprobe bonding

# Create bond in balance-tlb mode
ip link add bond0 type bond mode balance-tlb

# Set MII monitoring interval
ip link set bond0 type bond miimon 100

# Add slave interfaces
ip link set eth0 down
ip link set eth1 down
ip link set eth0 master bond0
ip link set eth1 master bond0

# Bring up the bond
ip link set bond0 up
ip addr add 192.168.1.100/24 dev bond0
ip route add default via 192.168.1.1
```

## Verify Balance-TLB

```bash
cat /proc/net/bonding/bond0
# Bonding Mode: adaptive transmit load balancing
# Primary Slave: None
# Currently Active Slave: eth0
```

## Set Primary Interface

```bash
# Set a primary slave (preferred receive interface)
echo eth0 > /sys/class/net/bond0/bonding/primary
```

## Persistent Configuration

```yaml
# Netplan
bonds:
  bond0:
    interfaces: [eth0, eth1]
    addresses: [192.168.1.100/24]
    parameters:
      mode: balance-tlb
      mii-monitor-interval: 100
      primary: eth0
```

```bash
# nmcli (RHEL)
nmcli connection add \
    type bond \
    con-name bond-tlb \
    ifname bond0 \
    bond.options "mode=balance-tlb,miimon=100,primary=eth0"
```

## Monitor Load Distribution

```bash
# Watch per-slave statistics to see load distribution
watch -n 2 "cat /proc/net/bonding/bond0"

# Check interface statistics
ip -s link show eth0
ip -s link show eth1
```

## Balance-TLB Advantages

- No switch support required
- Works with any standard switch
- Dynamic load balancing based on actual interface utilization
- Automatic failover

## Limitations

- Receive traffic is not load-balanced (only transmit)
- Lower aggregate receive throughput compared to mode 4 (LACP)
- Requires kernel support for load monitoring

## Conclusion

Balance-TLB provides outbound load balancing without any switch configuration, making it easy to deploy in any environment. The kernel dynamically distributes transmit traffic based on load, while receive traffic flows on the active slave. For full bidirectional load balancing without switch LACP support, use mode 6 (balance-ALB) which adds adaptive receive balancing.
