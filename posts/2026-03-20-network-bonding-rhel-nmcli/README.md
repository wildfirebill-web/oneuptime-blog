# How to Configure Network Bonding on RHEL with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RHEL, Network Bonding, nmcli, NetworkManager, Link Aggregation, High Availability

Description: Configure network interface bonding on Red Hat Enterprise Linux using nmcli to create persistent bond connections for redundancy or aggregation.

## Introduction

On RHEL, CentOS, and Fedora, nmcli (NetworkManager CLI) is the standard tool for creating bonds. nmcli creates persistent connection profiles that NetworkManager applies automatically at boot, handling failover and monitoring automatically.

## Prerequisites

- RHEL 7, 8, or 9 (or CentOS/Fedora equivalent)
- NetworkManager running
- Two or more physical interfaces
- Root access

## Step 1: Create the Bond Interface

```bash
# Create a bond master connection (active-backup mode)

nmcli connection add \
    type bond \
    con-name bond0 \
    ifname bond0 \
    bond.options "mode=active-backup,miimon=100"
```

## Step 2: Add Slave Interfaces

```bash
# Add eth0 as a bond slave
nmcli connection add \
    type ethernet \
    con-name bond0-slave-eth0 \
    ifname eth0 \
    master bond0

# Add eth1 as a bond slave
nmcli connection add \
    type ethernet \
    con-name bond0-slave-eth1 \
    ifname eth1 \
    master bond0
```

## Step 3: Assign an IP Address

```bash
# Add static IP to the bond
nmcli connection modify bond0 \
    ipv4.addresses "192.168.1.100/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "8.8.8.8" \
    ipv4.method manual

# Or use DHCP
nmcli connection modify bond0 ipv4.method auto
```

## Step 4: Activate the Bond

```bash
# Activate slave connections first
nmcli connection up bond0-slave-eth0
nmcli connection up bond0-slave-eth1

# Activate the bond master
nmcli connection up bond0
```

## Verify Bond Status

```bash
# Check bond details
nmcli connection show bond0

# Check interface status
ip addr show bond0

# View bonding details from the kernel
cat /proc/net/bonding/bond0
```

## LACP (802.3ad) Bond

```bash
nmcli connection add \
    type bond \
    con-name bond-lacp \
    ifname bond0 \
    bond.options "mode=802.3ad,lacp_rate=fast,miimon=100,xmit_hash_policy=layer3+4"
```

## Modify Bond Options

```bash
# Change bond mode (requires deactivation first)
nmcli connection down bond0
nmcli connection modify bond0 bond.options "mode=balance-alb,miimon=100"
nmcli connection up bond0
```

## Set Primary Interface

```bash
# Set eth0 as the primary (preferred active interface in active-backup mode)
nmcli connection modify bond0 \
    bond.options "mode=active-backup,miimon=100,primary=eth0"
```

## Conclusion

nmcli on RHEL creates bonds in three steps: create the bond master, add slave connections, then assign the IP. nmcli configurations are persistent by default. The `bond.options` parameter accepts comma-separated bonding driver options from `/proc/net/bonding/` documentation. Always verify with `cat /proc/net/bonding/bond0` after setup.
