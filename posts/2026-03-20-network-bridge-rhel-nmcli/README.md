# How to Create a Network Bridge on RHEL Using nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RHEL, Network Bridge, nmcli, NetworkManager, Networking, KVM, Virtualization

Description: Create a persistent network bridge on Red Hat Enterprise Linux using nmcli to add a physical interface to a bridge for KVM virtual machine networking.

## Introduction

On RHEL, nmcli is the standard tool for creating network bridges through NetworkManager. nmcli creates persistent bridge connections that are automatically restored at boot. This guide creates a bridge suitable for KVM hypervisor use.

## Step 1: Create the Bridge Connection

```bash
# Create the bridge master
nmcli connection add \
    type bridge \
    con-name br0 \
    ifname br0

# Assign a static IP to the bridge
nmcli connection modify br0 \
    ipv4.method manual \
    ipv4.addresses "192.168.1.100/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "8.8.8.8"
```

## Step 2: Add Physical Interface to the Bridge

```bash
# Create a slave connection for eth0
nmcli connection add \
    type bridge-slave \
    con-name br0-slave-eth0 \
    ifname eth0 \
    master br0
```

## Step 3: Activate the Bridge

```bash
# Bring up the bridge and slave
nmcli connection up br0-slave-eth0
nmcli connection up br0

# Verify
ip addr show br0
bridge link show
```

## Disable STP (Optional for KVM)

```bash
# Disable STP for KVM environments (no bridge loops)
nmcli connection modify br0 bridge.stp no
nmcli connection up br0
```

## Configure Bridge with DHCP

```bash
nmcli connection add \
    type bridge \
    con-name br0 \
    ifname br0 \
    ipv4.method auto

nmcli connection add \
    type bridge-slave \
    con-name br0-eth0 \
    ifname eth0 \
    master br0

nmcli connection up br0-eth0
nmcli connection up br0
```

## Verify the Bridge

```bash
# Show bridge details
nmcli connection show br0

# Show bridge ports
bridge link show

# Check IP assignment
ip addr show br0

# Show forwarding database
bridge fdb show br br0
```

## Delete a Bridge

```bash
# Deactivate and delete the connections
nmcli connection down br0
nmcli connection delete br0
nmcli connection delete br0-slave-eth0
```

## Conclusion

nmcli creates persistent bridges on RHEL in two steps: create the bridge master with `type bridge`, then add slave connections with `type bridge-slave`. The bridge replaces the physical interface as the network endpoint. Disable STP (`bridge.stp no`) for KVM hypervisors where bridge loops are not a concern, eliminating the 30-second forwarding delay.
