# How to Configure Network Bonding on Ubuntu with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ubuntu, Network Bonding, Netplan, High Availability, Link Aggregation, Networking

Description: Set up network interface bonding on Ubuntu using Netplan YAML configuration to provide link redundancy or aggregation with persistent configuration.

## Introduction

Network bonding aggregates multiple physical interfaces into a single logical interface for redundancy or increased throughput. Ubuntu's Netplan provides a clean YAML-based way to configure bonding that is persistent across reboots and supports all Linux bonding modes.

## Prerequisites

- Ubuntu 18.04 or later
- Two or more physical network interfaces (e.g., eth0, eth1)
- Netplan installed (default on Ubuntu)
- Root access

## Active-Backup Bond (Mode 1) — High Availability

Active-backup keeps one interface active; the other takes over on failure:

```yaml
# /etc/netplan/01-bond.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false
    eth1:
      dhcp4: false

  bonds:
    bond0:
      interfaces:
        - eth0
        - eth1
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      parameters:
        mode: active-backup
        primary: eth0
        mii-monitor-interval: 100
```

## LACP Bond (Mode 4) — Link Aggregation

Requires switch support for LACP (802.3ad):

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false
    eth1:
      dhcp4: false

  bonds:
    bond0:
      interfaces: [eth0, eth1]
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

## Round-Robin Bond (Mode 0) — Load Balancing

Sends packets to each interface in turn:

```yaml
bonds:
  bond0:
    interfaces: [eth0, eth1]
    addresses: [192.168.1.100/24]
    parameters:
      mode: balance-rr
      mii-monitor-interval: 100
```

## Apply and Verify

```bash
# Apply the configuration
netplan apply

# Verify bond is active
ip addr show bond0
cat /proc/net/bonding/bond0

# Check which interface is active
cat /proc/net/bonding/bond0 | grep "Currently Active"
```

## Test Failover (Active-Backup)

```bash
# Disable the active interface to test failover
ip link set eth0 down

# Verify bond continues on eth1
cat /proc/net/bonding/bond0

# Re-enable eth0
ip link set eth0 up
```

## Conclusion

Netplan makes bond configuration clean and declarative on Ubuntu. Define slave interfaces in `ethernets` without IPs, then declare the bond in `bonds` with the mode and monitoring parameters. Apply with `netplan apply` and verify with `/proc/net/bonding/bond0`. Configuration persists across reboots automatically.
