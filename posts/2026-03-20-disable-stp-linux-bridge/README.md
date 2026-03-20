# How to Disable STP on a Linux Bridge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bridge, STP, Spanning Tree Protocol, KVM, Networking, Performance

Description: Disable Spanning Tree Protocol on a Linux bridge to eliminate the 30-second forwarding delay and improve network startup time in loop-free topologies.

## Introduction

STP is enabled by default on some Linux bridge configurations but adds up to 30 seconds of delay before a bridge port starts forwarding traffic. In environments like KVM hypervisors where there are no loops, disabling STP improves VM startup time and reduces unnecessary overhead.

## Check Current STP State

```bash
# Check STP state (0 = disabled, 1 = enabled)

cat /sys/class/net/br0/bridge/stp_state

# Or using ip
ip -d link show br0 | grep bridge
```

## Disable STP at Runtime

```bash
# Disable STP on the bridge
ip link set br0 type bridge stp_state 0

# Verify
cat /sys/class/net/br0/bridge/stp_state
# 0

# Also set forward_delay to 0 for instant forwarding
echo 0 > /sys/class/net/br0/bridge/forward_delay
```

## Using brctl (Legacy Tool)

```bash
# Install bridge-utils
apt install bridge-utils

# Disable STP with brctl
brctl stp br0 off

# Verify
brctl show br0
# STP enabled: no
```

## Persistent with Netplan

```yaml
bridges:
  br0:
    interfaces: [eth0]
    addresses: [192.168.1.100/24]
    parameters:
      stp: false
      forward-delay: 0
```

## Persistent with nmcli

```bash
nmcli connection modify br0 bridge.stp no
nmcli connection up br0
```

## Persistent with systemd-networkd

```ini
# /etc/systemd/network/10-br0.netdev
[NetDev]
Name=br0
Kind=bridge

[Bridge]
STP=no
```

## Persistent with /etc/network/interfaces

```bash
auto br0
iface br0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0     # Forward delay = 0
```

## Effect of Disabling STP

| With STP | Without STP |
|---|---|
| 30 second startup delay | Instant forwarding |
| Loop protection | No loop protection |
| BPDU packets overhead | No BPDU traffic |
| Suitable for redundant topologies | Suitable for loop-free topologies |

## When It Is Safe to Disable STP

- Single bridge with one uplink (physically impossible to create a loop)
- KVM hypervisor with VM tap interfaces only
- Container networking bridges
- Point-to-point bridge connections

## Conclusion

Disable STP with `ip link set br0 type bridge stp_state 0` when your bridge topology has no loops. This eliminates the 30-second startup delay and removes BPDU overhead. Always set `forward_delay` to 0 alongside disabling STP. Never disable STP in topologies with redundant physical paths - doing so will cause broadcast storms.
