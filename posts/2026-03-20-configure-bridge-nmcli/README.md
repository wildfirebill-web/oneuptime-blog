# How to Configure a Bridge with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Bridge, Networking, KVM

Description: Configure a Linux network bridge using nmcli, including creating the bridge device, adding physical interface members, and assigning an IP address.

## Introduction

NetworkManager supports bridge creation through nmcli. A bridge requires a master connection for the bridge device and slave connections for each physical interface member. This is commonly used for KVM virtual machine networking.

## Step 1: Create the Bridge Master

```bash
# Create a bridge connection
nmcli connection add \
    type bridge \
    con-name "br0" \
    ifname br0

# Assign a static IP to the bridge
nmcli connection modify "br0" \
    ipv4.method manual \
    ipv4.addresses "192.168.1.10/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "8.8.8.8"
```

## Step 2: Add a Physical Interface as a Slave

```bash
# Add eth0 as a bridge member
nmcli connection add \
    type ethernet \
    con-name "br0-slave-eth0" \
    ifname eth0 \
    master br0
```

## Step 3: Activate the Bridge

```bash
# Bring up the slave, then the bridge
nmcli connection up "br0-slave-eth0"
nmcli connection up "br0"
```

## Verify Bridge Configuration

```bash
# Show bridge interfaces
ip link show type bridge
bridge link show

# Check IP on bridge
ip addr show br0

# Show bridge details
nmcli connection show "br0"
```

## Configure STP on the Bridge

```bash
# Enable STP
nmcli connection modify "br0" \
    bridge.stp yes \
    bridge.forward-delay 4 \
    bridge.hello-time 2

# Disable STP
nmcli connection modify "br0" \
    bridge.stp no

nmcli connection up "br0"
```

## Bridge with DHCP

```bash
nmcli connection add \
    type bridge \
    con-name "br0" \
    ifname br0 \
    ipv4.method auto
```

## Add Multiple Slaves

```bash
# Add second physical interface as bridge member
nmcli connection add \
    type ethernet \
    con-name "br0-slave-eth1" \
    ifname eth1 \
    master br0

nmcli connection up "br0-slave-eth1"
```

## Delete a Bridge

```bash
nmcli connection down "br0"
nmcli connection delete "br0-slave-eth0"
nmcli connection delete "br0"
```

## Conclusion

nmcli bridge setup requires a master connection (`type bridge`) and slave connections for each member physical interface (`master br0`). IP configuration goes on the bridge master. STP settings are configured with `bridge.stp` and `bridge.forward-delay`. Verify with `bridge link show` and `ip addr show br0`.
