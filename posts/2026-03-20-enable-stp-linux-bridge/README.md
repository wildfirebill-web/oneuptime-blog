# How to Enable STP on a Linux Bridge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bridge, STP, Spanning Tree Protocol, Networking, Loop Prevention

Description: Enable Spanning Tree Protocol (STP) on a Linux bridge to prevent network loops when multiple bridges or redundant paths exist in your network topology.

## Introduction

Spanning Tree Protocol (STP) prevents network loops in bridged networks by blocking redundant paths and only allowing one active path at a time. STP is necessary when you have multiple switches or bridges with redundant connections. Without STP, a loop causes broadcast storms that can bring down the network.

## Enable STP on an Existing Bridge

```bash
# Enable STP on the bridge
ip link set br0 type bridge stp_state 1

# Verify STP is enabled
cat /sys/class/net/br0/bridge/stp_state
# 1 = enabled

# Or check bridge details
bridge link show
```

## STP States and Timing

After enabling STP, ports go through multiple states before forwarding:

| State | Duration | Description |
|---|---|---|
| Blocking | Until elected | Won't forward, won't learn |
| Listening | 15 seconds | Participates in STP election |
| Learning | 15 seconds | Learns MAC addresses, no forwarding |
| Forwarding | Active | Fully active, forwards traffic |

Total delay: up to 30 seconds before traffic flows!

## Configure STP Timers

```bash
# Set bridge priority (lower = more likely to be root bridge)
echo 4096 > /sys/class/net/br0/bridge/priority

# Set forward delay (seconds in listening + learning states)
echo 4 > /sys/class/net/br0/bridge/forward_delay

# Set hello time (seconds between STP BPDUs)
echo 2 > /sys/class/net/br0/bridge/hello_time

# Set max age (time to hold BPDU info)
echo 12 > /sys/class/net/br0/bridge/max_age
```

## Check STP State via brctl

```bash
# Install bridge-utils
apt install bridge-utils

# Show STP state and bridge info
brctl showstp br0

# Example output shows each port's STP state
```

## Configure with Netplan

```yaml
bridges:
  br0:
    interfaces: [eth0, eth1]
    addresses: [192.168.1.100/24]
    parameters:
      stp: true
      forward-delay: 4
      hello-time: 2
      max-age: 12
      priority: 4096
```

## Rapid STP (RSTP)

The Linux bridge implements STP (802.1D) which has slow convergence. For faster convergence (1-2 seconds), consider using Rapid STP (RSTP/802.1w). The Linux kernel bridge does not fully implement RSTP, but you can use Open vSwitch for RSTP support.

## When to Use STP

- Multiple physical bridges/switches connected in a loop topology
- Redundant uplinks from a bridge to the network
- Any setup where a loop is physically possible

## When NOT to Use STP

- Single bridge with single uplink (no loops possible)
- KVM hypervisor with VMs (no loops, just overhead)
- Environments where the 30s startup delay is unacceptable

## Conclusion

Enable STP with `ip link set br0 type bridge stp_state 1` when your bridge is part of a topology with redundant paths. STP prevents broadcast storms but adds 30 seconds of startup delay. Tune `forward_delay` down to 4 seconds (minimum recommended) to reduce startup time while maintaining loop protection.
